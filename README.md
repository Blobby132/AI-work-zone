# AI-work-zone

## claude-code-dashboard

A local Grafana dashboard for Claude Code usage — spend, tokens, cache hit
rate, code churn and permission decisions, fed by Claude Code's OpenTelemetry
export.

```bash
cd claude-code-dashboard && make up
```

See [`claude-code-dashboard/README.md`](claude-code-dashboard/README.md) for
setup, and read the "One thing worth understanding" section before editing a
panel — the queries deliberately avoid `rate()` and `increase()`.
