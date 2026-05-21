# Service Level Objectives — STANDARD Platform

**Status:** RATIFIED (2026-05-06, alex)
**Last reviewed:** 2026-05-06
**Owners:** Alex (engineering) + Tiffy (product)
**Review cadence:** quarterly, or after any sustained budget burn
**Related:** [`OPERATIONAL_RUNBOOK.md`](./OPERATIONAL_RUNBOOK.md), [`grafana/alerts/rules.yml`](../../grafana/alerts/rules.yml), [`monitoring-alerting-plan.md`](../../../../Obsidian/iowire/Architecture/monitoring-alerting-plan.md)

## Purpose

Set explicit reliability targets for STANDARD's user-facing surfaces so
operators know when to escalate, on-call has a budget to work against,
and product knows what's worth spending engineering time on.

These are **launch-phase** values — conservative enough to be defensible
at small scale, tight enough that we notice if reliability slips. Tighten
in the next quarterly review once we have ≥30 days of production traffic
data.

## Service tiers

We classify user-facing flows into three tiers. Tier determines target
availability and how tight the latency SLO is.

| Tier | Definition | Examples |
|------|------------|----------|
| **Critical** | User cannot complete the core money-moving flow without it | `/login`, payment-intent creation, Stripe webhook receipt, `/submit-music` POST, queue join |
| **Important** | Degrades the experience but doesn't block revenue | `/` (homepage), host dashboard, `/queue`, leaderboard, search |
| **Best-effort** | Nice-to-have; failure is recoverable on retry | analytics, sync of off-platform engagement, push notifications, social-post embeds |

## Service Level Indicators (SLIs)

| SLI | Definition | Source |
|-----|------------|--------|
| **Availability** | `1 - (sum(rate(http_errors_total[5m])) / sum(rate(http_requests_total[5m])))`, measured over a rolling window | Prometheus / Grafana — `High5xxRate` rule reuses the numerator |
| **Critical-path latency** | p95 wall-clock for the named endpoint, steady-state (excluding the first 60s after a deploy) | Sentry Performance + Grafana `http_request_duration_seconds_bucket` |
| **Webhook success rate** | `1 - (rate(webhook_processed_total{status="failed"}[1h]) / rate(webhook_processed_total[1h]))` excluding intentional `signature_invalid` rejections | App-emitted custom metric; cross-reference Stripe Dashboard event log |
| **External reachability** | Synthetic `/health` probe success rate from ≥2 regions (Atlanta + Frankfurt) | Grafana Synthetic Monitoring — `ExternalReachabilityFailed` rule consumes this |

## Service Level Objectives (SLOs)

### Availability — monthly

| Tier | Target | Allowable downtime / month |
|------|--------|----------------------------|
| Critical | **99.5%** | 3h 39m |
| Important | 99.0% | 7h 18m |
| Best-effort | 95.0% | 36h 30m |

99.5% monthly was chosen over the 99.5%-weekly baseline in earlier sprint
notes — weekly is too tight for a single-EC2 stack with manual ops. We
revisit after Q3 if the budget consistently underburns.

### Latency — steady-state p95

| Path | Tier | Target p95 | Notes |
|------|------|------------|-------|
| `/` homepage (anon) | Important | < 1s | Phase 1 query cache (`585eedcc`) is the load-bearing optimization |
| `/login` POST | Critical | **< 5s** | Argon2id is intentionally slow; B5e investigation may move this lower |
| `/api/payments/create-payment-intent` | Critical | < 5s | Stripe API round-trip dominates |
| `/api/webhooks/stripe` POST | Critical | < 2s | Must ack within Stripe's 30s timeout; allow retry budget |
| `/submit-music` POST | Critical | < 3s | DB write + queue insert |
| `/queue` GET | Important | < 1s | Cached at query layer |
| `/leaderboard` GET | Important | < 2s | 60s TTL cache |

### Error rate

- **5xx rate < 1%** measured per 5-minute window across all critical-tier
  paths. The Grafana `High5xxRate` alert fires at **5%** — that's
  emergency-pager territory, not the SLO. The SLO is the line we should
  not be crossing routinely.

### Webhook reliability

- **Stripe webhook success rate ≥ 99%** measured over rolling 24h,
  excluding intentional `signature_invalid` rejections (which are working
  as designed). Below this, escalate per `OPERATIONAL_RUNBOOK.md` →
  `Issue 6` (Authentication Not Working) and the Stripe-cutover runbook.

### External reachability

- **/health probe success ≥ 99.5% per 24h per region.** Two-region setup
  (Atlanta + Frankfurt). Single-region failures may indicate AWS routing
  weather, not platform-down; multi-region failure pages immediately.

## Error budget

The error budget is the inverse of the availability SLO — for Critical
tier, **0.5% of the month** = 3h 39m. Operationally:

- **Burn rate < 1×** (consuming budget at the SLO pace): no action; this is normal
- **Burn rate 1-2× sustained for >24h:** investigate; pause non-critical deploys
- **Burn rate >2× sustained for >1h:** treat as incident; freeze deploys until budget recovers

Burn-rate alerts are not yet wired (deferred from B3 phase 1). Phase 2 of
the alerting plan will add them — see
[`monitoring-alerting-plan.md`](../../../../Obsidian/iowire/Architecture/monitoring-alerting-plan.md).

## What's NOT covered by this SLO (yet)

- Background Celery jobs (digests, score calculation, social sync). These
  have eventual-consistency semantics; user-visible failure is captured
  by the Critical-tier path SLOs that depend on them.
- Off-platform analytics correctness (TikTok metrics ingest, Spotify
  follower counts). Best-effort tier; staleness up to 24h is acceptable.
- LiveKit session reliability — measured separately in the LiveKit
  ops dashboard; SLO will be drafted when on-platform streaming becomes
  Critical-tier (today it's an alternative to off-platform sessions).

## Measurement and reporting

- **Real-time:** Grafana dashboards on celtics — `dashboards/standard-platform-overview.json`
- **Monthly review:** generate a markdown summary at `tasks/slo-monthly-<YYYY-MM>.md` showing actual vs. target per tier, trend vs. prior month, and burn-rate windows. Pin to vault `Compliance/Evidence/`.
- **Quarterly review:** revisit thresholds and tier assignments based on traffic, incident history, and product priority changes. Update this doc.

## Ratification log

- **2026-05-06** — Drafted by Alex; baseline values reflect cloud-architect-sre's Sprint 107 readiness assessment + B5 load-test findings.
- **2026-05-06** — **Ratified by Alex** (sole sign-off; per `.claude-shared/rules/operators.md` Alex and Tiffy are full peers and either operator's sign-off is binding). Sprint 107 exit criterion #4 flips YELLOW → GREEN. Tiffy may revise during the next quarterly review or earlier if product priorities change.
- _Future entries: append on each quarterly review._
