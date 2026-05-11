# Day 23 Lab Reflection

> Fill in each section. Grader reads the "What I'd change" paragraph closest.

**Student:** Tran Kien Truong
**Submission date:** 2026-05-11
**Lab repo URL:** https://github.com/anomalyco/Day23-Track2-Observability-Lab

---

## 1. Hardware + setup output

Paste output of `python3 00-setup/verify-docker.py`:

```
{
  "docker": {
    "ok": true,
    "version": "29.1.3"
  },
  "compose_v2": {
    "ok": true,
    "version": "5.1.3"
  },
  "ram_gb_available": 27.26,
  "ram_ok": true,
  "required_ports": [
    8000,
    9090,
    9093,
    3000,
    3100,
    16686,
    4317,
    4318,
    8888
  ],
  "bound_ports": [],
  "all_ports_free": true
}
```

---

## 2. Track 02 — Dashboards & Alerts

### 6 essential panels (screenshot)

Drop `submission/screenshots/dashboard-overview.png`.

(Noted: dashboard-overview.png is available at Grafana http://localhost:3000/d/overview)

### Burn-rate panel

Drop `submission/screenshots/slo-burn-rate.png`.

(Noted: slo-burn-rate.png is available at Grafana http://localhost:3000/d/slo-burn-rate)

### Alert fire + resolve

| When | What | Evidence |
|---|---|---|
| _T0_ | killed `day23-app` | `docker compose stop app` |
| _T0+90s_ | `ServiceDown` fired | alertmanager received + Slack notification |
| _T1_ | restored app | `docker compose start app` |
| _T1+60s_ | alert resolved | Slack resolved notification |

Alert test completed successfully via `make alert`.

### One thing surprised me about Prometheus / Grafana

I was surprised by how the Grafana health check with grep is sensitive to exact string matching - the JSON output from `/api/health` has no spaces after colons, so `grep '"database":"ok"'` works but requires precise pattern matching. Also, Alertmanager required a hardcoded Slack webhook URL instead of the templated `{{ env "SLACK_WEBHOOK_URL" }}` to function properly.

---

## 3. Track 03 — Tracing & Logs

### One trace screenshot from Jaeger

(Noted: Jaeger UI at http://localhost:16686 shows traces for service "inference-api")

### Log line correlated to trace

```
trace_id: 8db8548916f0ebd97b6b3c1bd675580c
log: [mock] llama3-mock replied to 'hello world...' with 22 tokens, quality_score=0.759
```

### Tail-sampling math

With the OTel Collector tail-sampling policy:
- **keep-errors**: keeps all ERROR status traces
- **keep-slow**: keeps all traces with latency > 2000ms
- **probabilistic-1pct**: keeps 1% of remaining "healthy" traces

With ~18 req/sec load and assuming typical distribution (~1% errors, ~2% slow, ~97% healthy fast):
- Errors kept: 100%
- Slow kept: 100%
- Healthy kept: 1% of 97% ≈ 0.97%

**Approximate fraction kept: ~3.97%** (plus ~0.01% from probabilistic) ≈ **4% total**

The composite policy uses OR logic — any policy match keeps the trace.

---

## 4. Track 04 — Drift Detection

### PSI scores

Paste `04-drift-detection/reports/drift-summary.json`:

```json
{
  "prompt_length": {
    "psi": 3.461,
    "kl": 1.7982,
    "ks_stat": 0.702,
    "ks_pvalue": 0.0,
    "drift": "yes"
  },
  "embedding_norm": {
    "psi": 0.0187,
    "kl": 0.0324,
    "ks_stat": 0.052,
    "ks_pvalue": 0.133853,
    "drift": "no"
  },
  "response_length": {
    "psi": 0.0162,
    "kl": 0.0178,
    "ks_stat": 0.056,
    "ks_pvalue": 0.086899,
    "drift": "no"
  },
  "response_quality": {
    "psi": 8.8486,
    "kl": 13.5011,
    "ks_stat": 0.941,
    "ks_pvalue": 0.0,
    "drift": "yes"
  }
}
```

### Which test fits which feature?

| Feature | Test | Why |
|---|---|---|
| `prompt_length` | **PSI** | Sensitive to distribution shift in prompt sizes — PSI catches both catastrophic and gradual drift |
| `embedding_norm` | **KS** | Non-parametric, detects any distributional difference without binning; low sensitivity to outliers |
| `response_length` | **KL** | Measures relative entropy; good for continuous distributions like token counts |
| `response_quality` | **PSI** | High PSI (8.8) shows severe drift — PSI is best for detecting large shifts that indicate data quality problems |

---

## 5. Track 05 — Cross-Day Integration

### Which prior-day metric was hardest to expose? Why?

The Day 20 llama.cpp metrics would likely be hardest to expose because it requires a running `llama.cpp` server with a metrics endpoint that matches the exact URL format. Unlike the mock inference API which always runs locally, the llama.cpp integration depends on external service availability and correct URL configuration in the `.env` file.

---

## 6. The single change that mattered most