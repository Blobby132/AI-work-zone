# Claude Code telemetry dashboard.
COMPOSE := docker compose

# The exact metric names the dashboard queries. Claude Code's own
# active_time metric really is named ...total, so a blanket "_total means
# suffixed" check gives a false alarm -- this list is the honest test.
EXPECTED := claude_code_session_count claude_code_lines_of_code_count \
            claude_code_pull_request_count claude_code_commit_count \
            claude_code_cost_usage claude_code_token_usage \
            claude_code_code_edit_tool_decision claude_code_active_time_total
CURL    := docker run --rm --network claude-code-dashboard_default curlimages/curl:latest -s --max-time 10

.PHONY: up down restart logs verify metrics reset help

help:
	@echo "make up      start the stack (Grafana on http://localhost:3000)"
	@echo "make down    stop it, keeping all data"
	@echo "make verify  check the stack is healthy and the metric names still match the dashboard"
	@echo "make metrics dump every claude_code series the collector is exposing"
	@echo "make logs    tail collector logs"
	@echo "make reset   stop and DELETE all stored metrics and logs"

up:
	$(COMPOSE) up -d
	@echo "Grafana:    http://localhost:3000"
	@echo "Collector:  OTLP gRPC on localhost:4317"
	@echo "Next: enable telemetry in Claude Code (see README), then 'make verify'."

down:
	$(COMPOSE) down

restart:
	$(COMPOSE) up -d --force-recreate

logs:
	$(COMPOSE) logs -f otel-collector

metrics:
	@$(CURL) http://otel-collector:8889/metrics | grep '^claude_code' || echo "No claude_code metrics yet - run a Claude Code session with telemetry enabled."

# The dashboard's queries are written against unsuffixed metric names, which
# depend on the collector's translation_strategy. That option has changed
# spelling before (add_metric_suffixes was silently dropped), so this target
# checks the live names rather than trusting the config.
verify:
	@echo "== containers =="
	@$(COMPOSE) ps --format 'table {{.Name}}\t{{.Status}}'
	@echo ""
	@echo "== prometheus scrape target =="
	@if $(CURL) 'http://prometheus:9090/api/v1/targets?state=active' \
	   | tr '{' '\n' | grep '"job":"claude-code"' | grep -q '"health":"up"'; then \
	     echo "  claude-code target: up"; \
	   else \
	     echo "  claude-code target: DOWN - the collector is not being scraped"; \
	   fi
	@echo ""
	@echo "== metric names (must match the set the dashboard queries) =="
	@$(CURL) http://otel-collector:8889/metrics > /tmp/cc-verify.txt 2>/dev/null || true
	@if ! grep -q '^claude_code' /tmp/cc-verify.txt 2>/dev/null; then \
	   echo "  No claude_code metrics are being exposed right now."; \
	   echo "  That is expected if no session has run in the last 10 minutes"; \
	   echo "  (metric_expiration in otel/collector.yaml). Run a Claude Code"; \
	   echo "  session with telemetry enabled, then re-run 'make verify'."; \
	 else \
	   grep '^# TYPE claude_code' /tmp/cc-verify.txt | sed 's/^# TYPE /  /'; \
	   for n in $(EXPECTED); do echo "$$n"; done > /tmp/cc-expected.txt; \
	   grep '^# TYPE claude_code' /tmp/cc-verify.txt | awk '{print $$3}' \
	     | grep -vxF -f /tmp/cc-expected.txt > /tmp/cc-unexpected.txt || true; \
	   echo ""; \
	   if [ -s /tmp/cc-unexpected.txt ]; then \
	     echo "  WARNING: these names are not what the dashboard queries:"; \
	     sed 's/^/    /' /tmp/cc-unexpected.txt; \
	     echo "  Expected e.g. claude_code_cost_usage, not"; \
	     echo "  claude_code_cost_usage_USD_total. Check translation_strategy"; \
	     echo "  in otel/collector.yaml - the option is version-sensitive."; \
	   else \
	     echo "  OK: names match what the dashboard queries."; \
	   fi; \
	   rm -f /tmp/cc-expected.txt /tmp/cc-unexpected.txt; \
	 fi
	@rm -f /tmp/cc-verify.txt

reset:
	$(COMPOSE) down -v
