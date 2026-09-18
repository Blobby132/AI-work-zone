# Claude Code dashboard

A local Grafana dashboard for your own Claude Code usage: what you spent, on
which models, how many tokens went to cache, how much code was written, and
which edits you rejected.

Claude Code can export OpenTelemetry. This is the other half — a collector to
receive it, Prometheus and Loki to store it, and a dashboard built against the
metric names Claude Code actually emits.

```
claude (any session)  --OTLP-->  collector  --scrape-->  Prometheus  --> Grafana
                                     \--------OTLP------>  Loki      --/
```

Nothing leaves your machine. Every port binds to `127.0.0.1`.

## Quick start

```bash
cd claude-code-dashboard
make up
```

Then turn on telemetry in Claude Code. Copy the `env` block from
[`claude/settings-snippet.json`](claude/settings-snippet.json) into your
`~/.claude/settings.json` (or a project `.claude/settings.json`):

```json
{
  "env": {
    "CLAUDE_CODE_ENABLE_TELEMETRY": "1",
    "OTEL_METRICS_EXPORTER": "otlp",
    "OTEL_LOGS_EXPORTER": "otlp",
    "OTEL_EXPORTER_OTLP_PROTOCOL": "grpc",
    "OTEL_EXPORTER_OTLP_ENDPOINT": "http://localhost:4317",
    "OTEL_METRIC_EXPORT_INTERVAL": "10000"
  }
}
```

Start a new Claude Code session — settings are read at startup, so an already
running session won't pick this up. Then:

```bash
make verify     # confirms the stack is healthy and the metric names still match
open http://localhost:3000
```

The dashboard is the Grafana home page. If you'd rather not edit settings,
`source claude/telemetry.env` before launching `claude` does the same thing for
one shell.

## What's on it

| Section | Shows |
| --- | --- |
| At a glance | Spend, sessions, tokens, active time, lines added/removed, commits, cache hit rate |
| Spend | Spend over time, and broken down by model, source (main vs subagent vs auxiliary), subagent, skill and MCP server |
| Tokens | Volume over time, mix across input / output / cacheRead / cacheCreation, and by model |
| Code and permissions | Lines changed, edit-tool accept/reject decisions, rejections by source and language, sessions by start type and terminal |
| Events | API errors over time, the API error stream, and the raw event stream |

The **Model** dropdown filters the cost and token panels.

## One thing worth understanding

Every total on this dashboard is computed as:

```promql
sum(max_over_time(claude_code_cost_usage[$__range]))
```

not with `rate()` or `increase()`. That is deliberate, and it is the single
thing to know before editing a panel.

Claude Code exports **cumulative counters from a process that lives for one
session**. A session starts, counts up, exits. Two consequences:

- `sum(claude_code_cost_usage)` would only count sessions still running.
- `increase()` and `rate()` return **zero**. They measure growth between
  samples, but the collector republishes a finished session's counter as a
  flat line, and Prometheus never observes the 0 → N jump because the first
  scrape already sees N. This was verified against live telemetry, not
  assumed — the first version of this dashboard used `increase()` and every
  time-series panel read zero.

Because each series is monotonic and uniquely labelled (`session_id` plus
model, query_source, and so on), the maximum a series reaches *is* that
session's final total. Summing those maxima is exact.

The caveat: a session straddling the edge of the selected time range is
counted in full, not pro-rated. Narrow ranges over long sessions read high.

The over-time panels bucket the same expression by `$__interval`, so they show
"the sessions active in this bucket, and what they had spent". A session
spanning two buckets appears in both — read them for shape, and the stat tiles
for exact totals.

## Checking the metric names

The dashboard queries `claude_code_cost_usage`. Out of the box the collector
would produce `claude_code_cost_usage_USD_total`, with the unit and counter
suffixes appended. `translation_strategy` in
[`otel/collector.yaml`](otel/collector.yaml) turns that off.

That option is version-sensitive: its predecessor, `add_metric_suffixes`, is
still accepted by collector 0.161.0 and **silently does nothing**. The image is
pinned for that reason. After any upgrade:

```bash
make verify
```

It compares the live metric names against the set the dashboard queries and
names anything that drifted.

## Attributing usage across people

Set custom resource attributes per person or team:

```bash
export OTEL_RESOURCE_ATTRIBUTES="department=engineering,team.id=platform"
```

Then add each key to `resource_constant_labels.included` in
`otel/collector.yaml`. It is a list of **exact attribute names, not patterns** —
an entry of `".*"` matches nothing and silently drops everything.

## Privacy

Metrics carry `user_email` and `organization_id` when you're authenticated.
That is fine for a stack on your own machine; think about it before pointing
this at a shared host.

Prompt text, model responses, tool commands and tool output are **not**
collected. The `OTEL_LOG_*` variables that enable them are commented out in
`claude/telemetry.env`. Turning them on ships that content to Loki in the
clear.

Grafana runs with anonymous admin access so the dashboard opens without a
login. That is only safe because every port is bound to `127.0.0.1` — if you
move this off localhost, remove the `GF_AUTH_ANONYMOUS_*` variables first.

## Troubleshooting

**Dashboard is empty.** Telemetry is read at session startup — restart
`claude`. Then `make verify`. If it reports no metrics, nothing arrived; check
`OTEL_EXPORTER_OTLP_ENDPOINT` points at `localhost:4317` and `make logs`.

**Panels show data but time-series are flat/zero.** Something reintroduced
`rate()` or `increase()`. See above.

**A breakdown panel is empty.** Subagent, skill and MCP panels only count
requests carrying those labels, and the rejection table only fills once you
reject an edit. Empty means it didn't happen, not that it's broken.

**Names look like `claude_code_cost_usage_USD_total`.** `make verify` will say
so. Check `translation_strategy`.

## Commands

```bash
make up        # start
make down      # stop, keep data
make verify    # health + metric-name check
make metrics   # dump every claude_code series the collector exposes
make logs      # tail collector logs
make reset     # stop and DELETE all stored metrics and logs
```

Metrics are kept for 365 days, events for 31.

## Versions

Pinned in `docker-compose.yml`, and what this was built and verified against:
OpenTelemetry Collector 0.161.0, Prometheus 3.14.0, Loki 3.7.8, Grafana 13.2.2,
Claude Code 2.1.276.
