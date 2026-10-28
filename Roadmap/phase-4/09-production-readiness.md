# 9. Production Readiness: Load Testing, Incidents and Runbooks

> Phase 4 · Week 12 · [← Phase index](README.md)

**Why this matters:** "Done" means **running reliably in production**, not merged. Before a launch, Middle developers are expected to know how much load their service handles, what happens when a dependency fails, how to roll back, and what to do when it pages at 3 AM.

---

## 9.1 Load testing with k6

```javascript
// load/checkout.js   Run: k6 run -e BASE_URL=https://staging.shop.example load/checkout.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '1m', target: 50 },    // ramp up to 50 virtual users
    { duration: '5m', target: 50 },    // steady state
    { duration: '1m', target: 200 },   // spike
    { duration: '2m', target: 0 },     // ramp down
  ],
  thresholds: {
    http_req_failed: ['rate<0.01'],                    // < 1% errors
    'http_req_duration{name:events}': ['p(95)<300'],   // p95 < 300 ms
  },
};

export default function () {
  const res = http.get(`${__ENV.BASE_URL}/api/events?page=1&pageSize=20`, { tags: { name: 'events' } });
  check(res, { 'status is 200': (r) => r.status === 200 });
  sleep(1);
}
```

| Test type | Question |
|---|---|
| **Load** | does it meet the SLO at expected peak traffic? |
| **Stress** | where does it break, and *how*? (gracefully with 429/503, or by falling over) |
| **Spike** | does it survive a sudden 10× jump (a ticket sale opening)? Does autoscaling react fast enough? |
| **Soak** | is it stable for hours? (memory leaks, connection-pool exhaustion, growing queues) |

**Load-testing rules:**

- Test a **production-like environment** with production-like data volumes. Ten rows in the database proves nothing.
- Watch the **server side** while testing: the RED dashboard, CPU, GC, thread pool, the DB connection pool, slow queries.
- Change **one thing at a time** and record the before and after numbers.
- **Never load test production or third-party APIs** without explicit approval. Use mocks for external providers.

---

## 9.2 A capacity and bottleneck checklist

| Bottleneck | Symptom | Typical fix |
|---|---|---|
| Database queries | slow spans on SQL; high DB CPU | indexes, projections, fewer round trips (Phase 2) |
| **DB connection pool exhausted** | timeouts acquiring connections | shorter transactions, async all the way, pool size tuning, PgBouncer |
| Thread-pool starvation | high latency with low CPU; growing queue length | remove sync-over-async (Phase 1, guide 3) |
| GC pressure | high % time in GC; Gen2 collections | reduce allocations, stream large payloads (Phase 5) |
| Downstream latency | wide spans on HTTP calls | timeouts, caching, async messaging |
| Lock contention | throughput flat while CPU is low | smaller critical sections, optimistic concurrency |
| Autoscaling lag | errors during spikes | higher minimum replicas, faster scale rules, queue-based load leveling |

---

## 9.3 Production readiness review

Go through this before a new service or major feature goes live:

| Area | Questions |
|---|---|
| **Observability** | dashboards (RED + dependencies), SLOs, burn-rate alerts, trace propagation, structured logs without PII |
| **Resilience** | timeouts on every outgoing call, retries in one layer, circuit breakers, graceful degradation, idempotency |
| **Scalability** | load test results vs expected peak; autoscaling rules; DB indexes for the new queries |
| **Deployment** | zero-downtime rollout, health probes, backward-compatible migrations, feature flag, **tested rollback** |
| **Security** | threat model done, authorization tests, secrets in a vault, dependency scan clean |
| **Data** | backups, restore tested, retention and GDPR deletion handled |
| **Operations** | runbooks for each alert, an on-call owner, dependencies' SLAs known |
| **Documentation** | README, architecture diagram, ADRs for the key decisions |

---

## 9.4 Runbooks

Every alert should link to a runbook. A runbook is a short, actionable page:

```markdown
# Runbook: Checkout error rate SLO burning

## Symptoms
Alert "checkout-slo-fast-burn". Users see "Payment failed, try again".

## Dashboards / queries
- Grafana: Shop / Checkout (RED + dependencies)
- Traces: service.name = shop-api AND http.route = /api/checkout AND status = error

## Diagnosis
1. Is the payments provider slow or erroring? (panel "payments.charge p95 / error rate")
   → check the provider status page
2. Was there a deployment in the last hour? (deployment annotations on the dashboard)
3. Database: connection pool usage, slow queries panel

## Mitigation
- Provider outage → enable the "PayLater" flag (App Configuration), post a status update
- Bad deployment → `az containerapp revision activate … --revision <previous>`, or re-run the previous pipeline
- DB saturation → scale up the read replica, enable the catalog cache flag

## Escalation
Payments team on-call (PagerDuty: payments-primary); provider support line in the vault: secret "payments-support"
```

---

## 9.5 Incident response basics

**During an incident:**

1. **Acknowledge** the page and declare an incident in the agreed channel.
2. Assign roles, even on a small team: an **incident lead** (coordinates), an **investigator**, and a **communicator** (status updates to stakeholders).
3. **Mitigate first, find the root cause later.** Roll back, flip a flag, scale up, fail over. Restoring service beats a perfect diagnosis.
4. Keep a **timeline** of what was observed and what was done, with timestamps.
5. Communicate regularly (for example, every 30 minutes), even when the update is "no change yet".

**After the incident: a blameless postmortem**

```markdown
# Postmortem: Checkout failures, 2027-09-14 14:05–14:52 UTC

## Impact
47 minutes; ~18% of checkouts failed; ~1,900 users affected; no data loss.

## Timeline
14:05 provider latency rises → 14:07 circuit breaker opens → 14:12 page → 14:20 diagnosis
→ 14:31 PayLater flag enabled → 14:52 provider recovered, flag disabled

## Root cause and contributing factors
- Provider incident (trigger)
- Our attempt timeout of 10 s held request threads too long (contributing)
- No alert on provider latency; we were paged only on the SLO burn (detection delay)

## What went well / what went poorly
…

## Action items (owner, due date)
- [ ] Reduce the payments attempt timeout to 2 s (Anna, 09-21)
- [ ] Add a provider latency alert with a runbook link (you, 09-21)
- [ ] Automate PayLater activation when the breaker is open > 5 min (Marco, 10-05)
```

**Blameless** means focusing on *systems and processes*, not people. "Why did the system let a 10 s timeout reach production?", not "who set it?". People only report problems honestly when they feel safe doing so.

---

## 9.6 Game days

Practice failure on purpose, in a test environment or carefully in production with approval:

- Kill a pod during load. Do requests fail?
- Block the database for 60 s. Do readiness probes remove the pods? Do they recover by themselves?
- Make the payments mock return 503s. Does the breaker open? Does the fallback work? Does the alert fire and link to the runbook?
- Roll back a deployment. How long does it really take?

You'll find gaps in a calm hour instead of during a real outage.

---

## Interview questions

<details><summary><b>1. How would you prepare a service for a big launch?</b></summary>

Set SLOs; run load, spike and soak tests against production-like data and fix bottlenecks; confirm timeouts, retries, breakers and fallbacks; check autoscaling and minimum replicas; get dashboards, alerts and runbooks ready; use a zero-downtime deployment with a tested rollback and feature flags; and do a game day for the key dependencies.
</details>

<details><summary><b>2. What's the first priority in an incident?</b></summary>

Mitigating user impact: roll back, toggle a flag, scale up or fail over. Then communicate. Root-cause analysis comes after service is restored, followed by a blameless postmortem with owned action items.
</details>

<details><summary><b>3. What makes a postmortem "blameless", and why does it matter?</b></summary>

It analyzes how the system and process allowed the failure, rather than blaming individuals. That encourages honest reporting and produces systemic fixes (guardrails, alerts, automation) instead of "be more careful".
</details>

---

## Exercises

1. Write a k6 script for a lab API, run load, spike and soak tests, find one bottleneck, fix it, and record before/after numbers ([Lab 12](10-labs.md#lab-12-load-test-and-game-day)).
2. Write a runbook for one alert you created in [guide 7](07-observability.md).
3. Run a mini game day on your lab deployment: kill pods, block the database, break the payments mock.
4. At work: read the last three postmortems (or write one for a recent incident you saw).

## Self-check

- [ ] I can write and interpret load, stress, spike and soak tests.
- [ ] I can run a production readiness review for my feature.
- [ ] I can write runbooks and a blameless postmortem.
- [ ] I know the incident roles and that mitigation comes first.

## Further reading

- k6 documentation (grafana.com/docs/k6): test types, thresholds, scenarios.
- Google SRE book: the chapters on managing incidents and postmortem culture.
- *Release It!* (Nygard): the production-readiness mindset.
