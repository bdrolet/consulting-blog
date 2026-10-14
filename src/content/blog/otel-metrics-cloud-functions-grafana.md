---
title: "Why Your Cloud Functions Metrics and Logs Vanish on the Way to Grafana"
pubDate: 2026-07-15
description: "Two non-obvious traps when wiring OpenTelemetry from GCP Cloud Functions (gen2, Python) into Grafana Cloud — cumulative counters that never produce a second sample, and a root logger that swallows your logs. Both come from the same root cause: short-lived invocations break assumptions built for long-running processes."
tags: ["opentelemetry", "gcp", "python", "observability", "grafana"]
---

I have a little email-triage system running on Cloud Functions gen2. Nothing exotic — a Pub/Sub message comes in, a Python function classifies the email, writes a row, moves on. I wired it up to Grafana Cloud with OpenTelemetry so I could watch it work: counters for emails processed, histograms for stage latency, the usual.

The metrics didn't show up. And when I went to figure out *why* they didn't show up, I discovered I couldn't see my own logs either.

Both bugs have the same root cause, and it's worth stating up front because it explains everything downstream: **OpenTelemetry metric export and Python logging are both designed for a long-running process. A Cloud Function is not one.** It wakes up, handles a request, and dies — usually well before any of the machinery that assumes "there will be a next time" gets a chance to run.

Here are the two traps, in the order I hit them.

## Trap 1: Cumulative counters need a second sample, and you never get one

OTel's `OTLPMetricExporter` uses **cumulative temporality** for counters by default. Every export sends the running total since the process started: `counter=1` after the first email, `counter=2` after the second, and so on.

On the query side, Prometheus `increase()` computes the change between the oldest and newest sample in the window. That's the key detail — it needs **at least two samples** to compute anything. One data point produces no rate, and the series just... doesn't appear.

In a long-running server this is a non-issue. The SDK spins up a daemon thread that exports every 60 seconds — `counter=0`, then `1`, then `2` — a steady stream of samples, and `increase()` has plenty to chew on. But my function handles one message and exits in a couple of seconds, long before the 60-second export interval fires. It emits exactly one cumulative data point (`counter=1`) and dies. Grafana gets a single sample, `increase()` shrugs, and the metric is invisible.

### The fix: flush before *and* after every invocation

The trick is to manufacture the second sample yourself. Flush a baseline at the **start** of the invocation, before doing any work — that's your first sample. Then flush again at the **end**, after incrementing the counter. Now Grafana sees a before/after pair and `increase()` returns a real number.

```python
@functions_framework.cloud_event
def process(cloud_event: CloudEvent) -> None:
    otel.flush()          # baseline export (counter=N, or 0 on cold start)
    try:
        run_pipeline(...)
        otel.emails_processed.add(1, {"category": "primary"})
    finally:
        otel.flush()      # post-work export (counter=N+1)
```

On a cold start the pre-flush exports `0` and the post-flush exports `1`. On a warm instance (Cloud Run reuses the container for several requests) the pre-flush exports whatever this instance has accumulated, `N`, and the post-flush exports `N+1`. Either way you get two samples with a delta between them.

A few things I got wrong before this worked:

**Flush the reader, not the provider.** Call `force_flush()` directly on the `PeriodicExportingMetricReader`, not on the `MeterProvider`. `MeterProvider.force_flush()` is supposed to delegate to its readers, but the delegation isn't reliable across SDK versions and it adds a failure mode for no benefit. Keep a reference to the reader at module scope and flush it directly:

```python
def flush() -> None:
    if _tracer_provider is not None:
        _tracer_provider.force_flush(timeout_millis=5_000)
    if _metric_reader is not None:
        _metric_reader.force_flush(timeout_millis=5_000)
```

**Don't reach for `shutdown()`.** It looks tempting — it does guarantee a final export — but `MeterProvider.shutdown()` can only be called once. On a warm instance, calling it after the first request would silence metrics for every request after that. `force_flush()` is the reusable one.

**Use a short timeout.** Five seconds, not thirty. `flush()` runs twice per invocation, so a generous timeout is a worst-case doubled on every request. In practice the OTLP HTTP POST finishes in under 200ms; 5s is pure headroom.

### Two things that'll waste an afternoon once metrics *do* show up

**Metric names get mangled by unit expansion.** The SDK expands unit abbreviations when it converts names to Prometheus format. So this:

```python
meter.create_histogram("inbox.stage.duration", unit="ms")
```

is queryable as `inbox_stage_duration_milliseconds_bucket` — dots become underscores, `ms` becomes `milliseconds`, and histograms get a `_bucket` suffix. Counters pick up `_total`. If you query the name you *defined*, you get nothing. When in doubt, list what actually landed:

```promql
{__name__=~"inbox.*"}
```

**Instant queries lie about sparse metrics.** This one fooled me twice. I have a metric that emits once a day, from a 5 AM sweep job. The container flushes a single sample and exits. An *instant* query — Grafana's default "now", or `/api/v1/query` with no timestamp — only returns series with a sample in roughly the last five minutes. Query it at noon and you get zero series, which reads exactly like "the metric is broken" when it's really "you're asking about the wrong moment." Query over a range, or pin the instant query to just after emission:

```promql
# range query over the day, not an instant "now"
sum by (action) (increase(inbox_sweep_actions_total[24h]))
```

## Trap 2: You debug all of that blind, because your logs never reach Cloud Logging

Here's where the story turns. To verify the metrics fix, I went to Cloud Logging to check for OTLP export errors — the SDK logs `Exception while exporting metrics` when export fails. There was nothing. Not "no errors" — *nothing*. None of my `logger.info` lines were there either. For any function.

My first guess was that logging simply wasn't configured. But it was — right at the top of `main.py`:

```python
logging.basicConfig(level=logging.INFO)
```

That should install a stderr handler at INFO. So my hypothesis was wrong, and it took staring at it to see why.

In the gen2 runtime, the function is served by gunicorn, and **gunicorn configures the root logger before it imports your `main.py`.** By the time `basicConfig` runs, the root logger already has handlers — and `basicConfig` is documented to do *nothing* when that's the case. It silently no-ops. No stderr handler gets installed.

So where do the logs go? In my setup, `otel.setup_telemetry()` attaches an OTLP `LoggingHandler` to the root logger. That handler works fine — my logs were flowing to Grafana Loki the whole time. They just never reached stderr, which is the only path to Cloud Logging. Every `logger.*` call, *including the SDK's own export errors*, went to Grafana and nowhere else. I was trying to debug the pipe using the very pipe that was broken.

### The fingerprint

The tell, if you ever see it: **direct writes to stdout/stderr show up in Cloud Logging, but `logger.*` calls don't.** In my case a library was printing tqdm progress bars straight to stderr, and those appeared fine — right next to a total absence of anything from Python's `logging`. That split (framework logging invisible, raw writes visible) is the signature of a missing stderr handler on the root logger.

### The fix is one keyword

```python
logging.basicConfig(
    level=logging.INFO,
    format="%(levelname)s %(name)s %(message)s",
    force=True,
)
```

`force=True` (Python 3.8+) removes whatever handlers are already on the root logger and installs a fresh stderr one. Put it at import time, *before* `setup_telemetry()` runs. The OTLP handler still gets added afterward, so you keep shipping logs to Grafana — you just also get them in Cloud Logging now. Both destinations.

Two last gotchas:

**It doesn't reproduce locally.** `setup_telemetry()` no-ops when there's no OTLP endpoint set, which is exactly how you run it on your laptop. With no OTLP handler attached, the root logger is empty when `basicConfig` runs, so it works perfectly. The bug only exists in production. A local "logs look fine" proves nothing.

**Look under the right resource type.** For gen2 functions, your *application* logs land under `resource.type="cloud_run_revision"`, not `cloud_function` — the latter carries only infrastructure logs. This is why the log console can look empty even after the fix:

```bash
gcloud logging read \
  'resource.type="cloud_run_revision" AND resource.labels.service_name="inbox-sweep"' \
  --project=your-project-id --freshness=10m \
  --format='value(timestamp, severity, textPayload)'
```

## The one-line takeaway

Both traps are the same mistake wearing different clothes: a tool that assumes a long-lived process, dropped into a runtime where processes live for two seconds. OTel's metric reader assumes it'll get to export again; Python's `basicConfig` assumes it's the first thing to touch the root logger. Neither is true on Cloud Functions gen2.

Flush before and after. Force the log config. Then go read the metrics you can finally see.
