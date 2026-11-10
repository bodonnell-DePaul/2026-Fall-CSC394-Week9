---
marp: true
theme: default
paginate: true
size: 16:9
title: Logging, Monitoring & Observability for Web Applications
style: |
  section { font-size: 26px; }
  h1 { color: #1f4e79; }
  h2 { color: #1f4e79; }
  code { font-size: 0.85em; }
  pre { font-size: 0.72em; }
  table { font-size: 0.8em; }
  .small { font-size: 0.75em; }
  section.lead h1 { font-size: 2.2em; }
  section.lead { text-align: center; }
---

<!-- _class: lead -->
<!-- _paginate: false -->

# Logging, Monitoring & Observability
### Knowing what your web application is *actually* doing in production

Web Application Development

<!--
Timing guide (≈75–85 min total):
- Part 0 Intro & motivation: 8 min
- Part 1 Logging: 20 min
- Part 2 Metrics & Monitoring: 15 min
- Part 3 Tracing & OpenTelemetry: 12 min
- Part 4 Alerting, SLOs, dashboards: 10 min
- Part 5 Frontend observability, privacy, wrap-up: 8 min
- Q&A / in-class exercise: 5–15 min (flexible)
-->

---

## Learning Objectives

By the end of this lecture you should be able to:

1. Explain the difference between **logging**, **monitoring**, and **observability**
2. Write **structured logs** with appropriate levels and context
3. Choose and instrument the right **metrics** (RED / USE / Golden Signals)
4. Describe **distributed tracing** and how a request is followed across services
5. Instrument a Node/Express app with **OpenTelemetry**
6. Define **SLIs, SLOs**, and write alerts that don't wake people up for nothing
7. Avoid common pitfalls: PII leaks, log spam, high-cardinality metrics

---

## Agenda

| Part | Topic | ~Time |
|---|---|---|
| 0 | Why this matters | 8 min |
| 1 | Logging | 20 min |
| 2 | Metrics & Monitoring | 15 min |
| 3 | Distributed Tracing & OpenTelemetry | 12 min |
| 4 | Alerting, SLOs & Dashboards | 10 min |
| 5 | Frontend, Privacy, Tooling & Wrap-up | 8 min |
| — | Exercise / Q&A | 5–15 min |

---

<!-- _class: lead -->

# Part 0
## Why this matters

---

## "It works on my machine" 🤷

Your app is deployed. A user emails:

> "The checkout page is really slow and sometimes says *Something went wrong*."

**Questions you must answer:**
- Is it slow for *everyone* or just this user?
- Since *when*? Did a deploy cause it?
- Which part is slow — DB? payment API? the browser?
- How many orders did we lose?

Without instrumentation you are **guessing**.

<!--
Ask the class: "What would you do right now to debug this?" Collect answers — most will say "look at the code" or "try to reproduce". Point out: production differs from local (data volume, concurrency, network, third parties).
-->

---

## Production is different from localhost

- Real traffic: **concurrency**, bursts, bots
- Real data: big tables, weird Unicode, empty fields
- Real networks: latency, timeouts, flaky third-party APIs
- Many instances: containers come and go; no single "server" to SSH into
- You **can't attach a debugger** to production

> You can't fix what you can't see.

---

## Three related ideas

| Term | Question it answers | Nature |
|---|---|---|
| **Logging** | *What happened?* (discrete events) | Record |
| **Monitoring** | *Is the system healthy right now?* (known questions) | Watch |
| **Observability** | *Why is it behaving this way?* (new, unknown questions) | Explore |

**Observability** = a property of a system: how well you can understand its internal state from the outputs it produces.

Monitoring tells you **that** something is wrong. Observability helps you find out **why**.

---

## The Three Pillars of Telemetry

1. **Logs** — timestamped records of discrete events
   `"payment failed for order 123: card_declined"`
2. **Metrics** — numeric measurements aggregated over time
   `http_requests_total{status="500"} = 42`
3. **Traces** — the path of one request through the system
   `GET /checkout → auth (5ms) → db (120ms) → stripe (900ms)`

Plus increasingly: **profiles**, **events**, **real-user monitoring (RUM)**

<!--
The pillars are complementary, not alternatives.
-->

---

## How they work together

```
 Alert fires:  "p95 latency on /checkout > 2s"        ← METRICS
       │
       ▼
 Dashboard:    latency spike started 14:02, only /checkout
       │
       ▼
 Trace:        stripe.charge span = 1.8s of the 2.0s    ← TRACES
       │
       ▼
 Logs (trace_id=abc123):
   WARN  stripe retry attempt=2 reason=timeout          ← LOGS
```

Metrics → **detect**. Traces → **locate**. Logs → **explain**.

---

<!-- _class: lead -->

# Part 1
## Logging

---

## What is a log?

An **append-only, timestamped record** of an event that happened in your program.

```js
console.log("User logged in");
```

That's a log… but a pretty useless one in production:
- Which user? When exactly? Which server?
- How do I search 10 million of these?
- Is it important or noise?

---

## Log Levels

| Level | Use for | Example |
|---|---|---|
| `TRACE`/`DEBUG` | Developer detail, off in prod | `cache lookup key=user:42` |
| `INFO` | Normal significant events | `order created id=981` |
| `WARN` | Unexpected but handled | `retrying payment, attempt 2` |
| `ERROR` | Operation failed | `payment failed: card_declined` |
| `FATAL` | App can't continue | `cannot connect to DB, exiting` |

**Rule of thumb:** if `ERROR` doesn't mean "someone should look at this", you're over-using it.

Set the level by **environment variable**: `LOG_LEVEL=info` in prod, `debug` locally.

---

## Unstructured vs Structured Logs

**Unstructured** (human-friendly, machine-hostile):
```
2026-11-10 14:02:11 User 42 bought 3 items for $59.97 in 812ms
```

**Structured** (JSON — machine-friendly, still readable):
```json
{"time":"2026-11-10T14:02:11.204Z","level":"info","msg":"order created",
 "userId":42,"itemCount":3,"totalCents":5997,"durationMs":812,
 "requestId":"7f3a…","service":"shop-api","env":"prod"}
```

Now you can query: *"all orders where durationMs > 500 in the last hour, grouped by service"*.

---

## Structured logging in Node.js with **pino**

```js
// logger.js
import pino from "pino";

export const logger = pino({
  level: process.env.LOG_LEVEL ?? "info",
  base: { service: "shop-api", env: process.env.NODE_ENV },
  redact: ["req.headers.authorization", "*.password", "*.cardNumber"],
});
```

```js
import { logger } from "./logger.js";

logger.info({ userId: 42, itemCount: 3 }, "order created");
logger.error({ err, orderId }, "payment failed");
```

Alternatives: **winston** (Node), `logging`/**structlog** (Python), **Serilog** (.NET), **SLF4J/Logback** (Java)

<!--
pino is fast because it writes JSON and defers formatting. Use pino-pretty locally for readable output: `node app.js | npx pino-pretty`.
-->

---

## Request logging middleware (Express)

```js
import express from "express";
import pinoHttp from "pino-http";
import { randomUUID } from "node:crypto";
import { logger } from "./logger.js";

const app = express();

app.use(pinoHttp({
  logger,
  genReqId: (req, res) => {
    const id = req.headers["x-request-id"] ?? randomUUID();
    res.setHeader("x-request-id", id);
    return id;
  },
}));

app.get("/orders/:id", async (req, res) => {
  req.log.info({ orderId: req.params.id }, "fetching order"); // includes reqId
  res.json(await getOrder(req.params.id));
});
```

Every line for a request now shares a **request ID** → you can reconstruct the whole story.

---

## Correlation IDs

```
Browser ──(x-request-id: 7f3a)──▶ API ──(x-request-id: 7f3a)──▶ Payments service
```

- Generate an ID at the **edge** (load balancer, gateway, or first service)
- **Propagate** it in headers to every downstream call
- **Include** it in every log line
- **Return** it to the client (show it on error pages: *"Error ref: 7f3a"*)

Support ticket: *"I got error 7f3a"* → search logs for `requestId=7f3a` → full story.

In modern systems, the OpenTelemetry **trace ID** plays this role (Part 3).

---

## What should you log?

✅ **Do log**
- Request start/finish: method, route, status, duration
- Important business events: signup, order placed, payment failed
- Errors **with stack traces** and context (IDs, inputs that matter)
- External calls: which service, duration, outcome
- Startup config (non-secret), version / git SHA

❌ **Don't log**
- Passwords, tokens, API keys, session cookies
- Full credit card numbers, SSNs, health data
- Entire request/response bodies by default
- Every iteration of a hot loop

---

## Logging errors well

```js
// ❌ Loses the stack trace and context
catch (e) { console.log("error: " + e.message); }

// ❌ Logs AND rethrows → duplicate noise at every layer
catch (e) { logger.error(e); throw e; }

// ✅ Add context, log once at the boundary
catch (err) {
  throw new PaymentError("charge failed", { cause: err, orderId });
}

// Central Express error handler
app.use((err, req, res, next) => {
  req.log.error({ err }, "unhandled error");
  res.status(500).json({ error: "Internal error", ref: req.id });
});
```

**Log at the boundary** where you handle the error — not at every level it passes through.

---

## Twelve-Factor App: logs are event streams

> "A twelve-factor app never concerns itself with routing or storage of its output stream."  — 12factor.net/logs

- App writes to **stdout/stderr** — that's it
- The platform (Docker, Kubernetes, Heroku, Azure App Service…) collects it
- A **log shipper** forwards to central storage

```
App → stdout → Collector (Fluent Bit / Vector / OTel Collector) → Store & Search
                                                    (Loki, Elasticsearch, Datadog, …)
```

Why not write to `app.log` files? Containers are ephemeral, disks fill up, files are per-machine.

---

## Log pitfalls

- **Log spam** — logging every cache hit at INFO → costs $$$ and hides signal
- **Inconsistent field names** — `userId`, `user_id`, `uid` → impossible to query
- **String concatenation** — `"user " + id + " did " + x` → can't filter by field
- **Leaking secrets/PII** — logs are often widely accessible and retained for a long time
- **Log injection** — user input containing newlines or fake JSON; structured loggers escape it
- **Synchronous logging in hot paths** — can slow your app

💡 Use **sampling** for very high-volume debug logs (e.g., log 1% of successful requests, 100% of errors).

---

## Quick check ✋

Which of these log lines is most useful in production? Why?

A. `console.log("here 2")`
B. `logger.info("Order failed")`
C. `logger.error({ orderId: 981, userId: 42, err, requestId }, "order failed")`
D. `logger.error({ order, user, req }, "order failed")`

<!--
Answer: C. D dumps entire objects (may include PII, huge, inconsistent). B lacks context. A is debugging residue.
-->

---

<!-- _class: lead -->

# Part 2
## Metrics & Monitoring

---

## What is a metric?

A **numeric measurement** over time, identified by a name and **labels** (dimensions).

```
http_requests_total{method="GET", route="/orders/:id", status="200"}  18342
http_request_duration_seconds_bucket{route="/checkout", le="0.5"}     912
process_resident_memory_bytes                                         1.3e8
```

- **Cheap** to store and query (numbers, not text)
- Great for **dashboards, trends, alerts**
- Lose individual detail — you know *how many* failed, not *which ones*

---

## Metric types

| Type | Behavior | Examples |
|---|---|---|
| **Counter** | Only goes up (resets on restart) | requests served, errors, orders placed |
| **Gauge** | Goes up and down | memory used, active connections, queue length |
| **Histogram** | Counts observations into buckets | request latency, response size |
| **Summary** | Client-side percentiles | (less common; prefer histograms) |

Ask: *"Do I care about the rate, the current value, or the distribution?"*

---

## Why averages lie: percentiles

100 requests: 95 take **50 ms**, 5 take **5,000 ms**

- **Average** = 297 ms → "looks fine"
- **p50 (median)** = 50 ms
- **p95** = 50 ms … **p99** = 5,000 ms 😬

Users experience the **tail**. A page that makes 20 API calls hits the p95 almost every load.

📏 Track **p50, p95, p99** latency — never just the mean.

---

## What should you measure? Frameworks

**RED** — for request-driven services (APIs, web servers)
- **R**ate — requests per second
- **E**rrors — failed requests per second
- **D**uration — latency distribution

**USE** — for resources (CPU, memory, disk, DB pool)
- **U**tilization · **S**aturation · **E**rrors

**Google SRE's Four Golden Signals**
- **Latency · Traffic · Errors · Saturation**

Plus **business metrics**: signups/min, orders/min, cart abandonment.

---

## Instrumenting Express with Prometheus (`prom-client`)

```js
import client from "prom-client";
client.collectDefaultMetrics(); // CPU, memory, event loop lag, GC…

const httpDuration = new client.Histogram({
  name: "http_request_duration_seconds",
  help: "HTTP request latency",
  labelNames: ["method", "route", "status"],
  buckets: [0.05, 0.1, 0.25, 0.5, 1, 2.5, 5],
});

app.use((req, res, next) => {
  const end = httpDuration.startTimer();
  res.on("finish", () =>
    end({ method: req.method, route: req.route?.path ?? "unknown",
          status: res.statusCode }));
  next();
});

app.get("/metrics", async (_req, res) => {
  res.set("Content-Type", client.register.contentType);
  res.end(await client.register.metrics());
});
```

---

## Pull vs Push

**Pull (Prometheus model)**
- App exposes `/metrics`; Prometheus **scrapes** it every N seconds
- Easy to tell if a target is down (scrape fails)

**Push (StatsD, OTLP, many SaaS)**
- App **sends** metrics to a collector
- Better for short-lived jobs, serverless, behind firewalls

Querying with **PromQL**:
```promql
# Error rate (%) over last 5 minutes
100 * sum(rate(http_request_duration_seconds_count{status=~"5.."}[5m]))
    / sum(rate(http_request_duration_seconds_count[5m]))

# p95 latency per route
histogram_quantile(0.95,
  sum by (le, route) (rate(http_request_duration_seconds_bucket[5m])))
```

---

## ⚠️ The cardinality trap

Each unique **combination of label values** = a separate time series.

```js
// ❌ DON'T: unbounded labels
httpDuration.labels({ userId: req.user.id, url: req.originalUrl })
// 100k users × 1M URLs = 💥 your metrics backend (and bill)

// ✅ DO: bounded labels
httpDuration.labels({ route: "/orders/:id", method: "GET", status: "200" })
```

- Use the **route template**, not the raw URL
- User IDs, emails, request IDs → belong in **logs/traces**, not metric labels
- Group status codes (`2xx`, `4xx`, `5xx`) if needed

---

## Health checks

```js
// Liveness: is the process alive? (don't check dependencies)
app.get("/healthz", (_req, res) => res.send("ok"));

// Readiness: can I serve traffic right now?
app.get("/readyz", async (_req, res) => {
  try {
    await db.query("SELECT 1");
    res.send("ready");
  } catch {
    res.status(503).send("db unavailable");
  }
});
```

- Used by load balancers / Kubernetes to route traffic and restart containers
- **Synthetic monitoring / uptime checks**: an external probe hits your site every minute from multiple regions

---

<!-- _class: lead -->

# Part 3
## Distributed Tracing & OpenTelemetry

---

## The problem tracing solves

Modern web apps are rarely one process:

```
Browser → CDN → API Gateway → Web API → Auth service
                                     ├→ Postgres
                                     ├→ Redis cache
                                     └→ Payments service → Stripe API
```

A request is slow. **Where** is the time going?
Logs from 6 services, metrics from 6 dashboards… stitching them by hand is painful.

---

## Traces and Spans

- A **trace** = the full journey of one request
- A **span** = one unit of work (has name, start, duration, attributes, status)
- Spans have **parent/child** relationships

```
Trace 4bf92f…  (total 2,040 ms)
├─ GET /checkout                         [web-api]    2,040 ms
│  ├─ auth.verifyToken                   [auth]          12 ms
│  ├─ SELECT cart_items …                [postgres]      85 ms
│  ├─ redis GET price:*                  [redis]          3 ms
│  └─ POST /charge                       [payments]   1,890 ms  ⚠️
│     └─ POST api.stripe.com/v1/charges  [http]       1,870 ms
```

Rendered as a **waterfall / Gantt chart** in tools like Jaeger, Zipkin, Tempo, Honeycomb.

---

## Context propagation

How does the Payments service know it's part of the same trace?

**W3C Trace Context** HTTP header:
```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             │  └──────────── trace-id ────────────┘ └─ parent span ─┘ └ flags (sampled)
             version
```

- Each service **reads** the incoming header, creates child spans, and **injects** it into outgoing calls
- Instrumentation libraries do this automatically for HTTP, gRPC, many DB clients and queues

---

## OpenTelemetry (OTel)

A **vendor-neutral, open standard** (CNCF) for generating and exporting telemetry.

- **APIs + SDKs** for many languages (JS, Python, Java, .NET, Go…)
- **Auto-instrumentation** for popular libraries (Express, http, pg, mysql, redis, fetch…)
- **OTLP** — a standard wire protocol for traces, metrics, logs
- **Collector** — receives, processes, and exports to any backend

```
App + OTel SDK ──OTLP──▶ OTel Collector ──▶ Jaeger / Tempo / Prometheus /
                                             Azure Monitor / Datadog / Honeycomb…
```

💡 Instrument once, switch backends without rewriting code.

---

## Auto-instrumenting a Node app

```bash
npm i @opentelemetry/sdk-node @opentelemetry/auto-instrumentations-node \
      @opentelemetry/exporter-trace-otlp-http
```

```js
// tracing.js — must load BEFORE your app code
import { NodeSDK } from "@opentelemetry/sdk-node";
import { getNodeAutoInstrumentations } from "@opentelemetry/auto-instrumentations-node";
import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-http";

const sdk = new NodeSDK({
  serviceName: "shop-api",
  traceExporter: new OTLPTraceExporter({ url: "http://localhost:4318/v1/traces" }),
  instrumentations: [getNodeAutoInstrumentations()],
});
sdk.start();
```

```bash
node --import ./tracing.js server.js
```

Every Express route, outgoing `fetch`/`http`, and DB query now produces spans. 🎉

---

## Custom spans for business logic

```js
import { trace, SpanStatusCode } from "@opentelemetry/api";
const tracer = trace.getTracer("shop-api");

async function placeOrder(cart, user) {
  return tracer.startActiveSpan("placeOrder", async (span) => {
    span.setAttributes({ "cart.items": cart.items.length, "user.tier": user.tier });
    try {
      const order = await chargeAndSave(cart, user);
      span.setAttribute("order.id", order.id);
      return order;
    } catch (err) {
      span.recordException(err);
      span.setStatus({ code: SpanStatusCode.ERROR, message: err.message });
      throw err;
    } finally {
      span.end();
    }
  });
}
```

---

## Connecting logs ↔ traces

Put the **trace ID** in every log line (OTel log instrumentation for pino/winston does this):

```json
{"level":"error","msg":"payment failed","orderId":981,
 "trace_id":"4bf92f3577b34da6a3ce929d0e0e4736","span_id":"00f067aa0ba902b7"}
```

Now in your tooling you can:
- Click a slow span → **see its logs**
- Click an error log → **jump to the full trace**

**Sampling:** tracing everything is expensive at scale
- *Head sampling*: decide at the start (e.g., keep 10%)
- *Tail sampling*: decide after (keep all errors + slow requests)

---

## Try it locally in 1 command

```bash
docker run --rm -p 16686:16686 -p 4318:4318 jaegertracing/all-in-one:latest
```

- Point your OTLP exporter at `http://localhost:4318`
- Open **http://localhost:16686** to view traces

Or a full local stack: **Grafana LGTM** (`grafana/otel-lgtm`) bundles Loki (logs), Grafana, Tempo (traces), Mimir/Prometheus (metrics) + an OTel Collector.

<!--
Live demo opportunity (5 min): run the Jaeger container, start the instrumented Express app, curl a few endpoints including one with an artificial delay, and show the waterfall.
-->

---

<!-- _class: lead -->

# Part 4
## Alerting, SLOs & Dashboards

---

## SLIs, SLOs, SLAs

- **SLI** (Indicator) — a measurement of user-visible behavior
  *"% of `/checkout` requests that succeed in < 500 ms"*
- **SLO** (Objective) — your internal target for an SLI
  *"99.5% over a rolling 30 days"*
- **SLA** (Agreement) — a contractual promise with consequences
  *"99.9% uptime or customer gets a credit"*

SLO should be **stricter than SLA** and **looser than 100%**.

---

## Error budgets

SLO 99.5% over 30 days → **0.5% may fail** → your **error budget**

| SLO | Allowed downtime / 30 days |
|---|---|
| 99% | ~7 h 12 m |
| 99.5% | ~3 h 36 m |
| 99.9% | ~43 m |
| 99.99% | ~4 m 19 s |

- Budget remaining → ship features, take risks
- Budget burned → slow down, focus on reliability

Turns "is it reliable enough?" into a **data-driven** conversation.

---

## Good alerts vs bad alerts

❌ **Bad**: *"CPU > 80% for 1 minute"*
- Is anything actually broken for users? Maybe not.
- Fires at 3 a.m., gets ignored → **alert fatigue**

✅ **Good**: *"Checkout error rate > 2% for 5 minutes"* or *"SLO burn rate is 10× normal"*
- **Symptom-based** (what users feel), not cause-based
- **Actionable** — someone can do something now
- Linked to a **runbook**: what to check, how to mitigate

**Page** a human only for urgent user impact. Everything else → ticket / dashboard / Slack.

---

## Example alert rule (Prometheus)

```yaml
groups:
- name: shop-api
  rules:
  - alert: HighErrorRate
    expr: |
      sum(rate(http_request_duration_seconds_count{status=~"5.."}[5m]))
        / sum(rate(http_request_duration_seconds_count[5m])) > 0.02
    for: 5m
    labels:
      severity: page
    annotations:
      summary: "shop-api 5xx error rate above 2%"
      runbook: "https://wiki.example.com/runbooks/shop-api-errors"
```

`for: 5m` prevents flapping on short blips.

---

## Dashboards that help

A good service dashboard, top to bottom:

1. **SLO status** & error budget remaining
2. **RED**: request rate, error rate, p50/p95/p99 latency
3. **Dependencies**: DB latency, cache hit rate, 3rd-party API errors
4. **Saturation**: CPU, memory, connection pool, queue depth
5. **Deploy markers** — vertical lines when new versions shipped

Avoid "wall of 60 graphs nobody understands". Each panel should answer a question.

---

## Incident workflow (where it all fits)

```
Detect     → alert fires (metrics / SLO burn / synthetic check)
Triage     → dashboard: what's affected? since when? recent deploy?
Investigate→ traces: where is time/error? logs: why?
Mitigate   → roll back, feature flag off, scale up
Resolve    → fix root cause
Learn      → blameless postmortem; add missing telemetry/alerts
```

**Blameless postmortems**: focus on *systems and processes*, not who to blame.

---

<!-- _class: lead -->

# Part 5
## Frontend, Privacy, Tooling & Wrap-up

---

## Don't forget the browser

Backend says 200 OK in 80 ms… but the user waited 6 seconds. Why?

**Real User Monitoring (RUM)**
- **Core Web Vitals**: LCP (loading), INP (interactivity), CLS (layout stability)
- JavaScript errors and unhandled promise rejections
- Slow/failed `fetch` calls from the client's perspective

```js
import { onLCP, onINP, onCLS } from "web-vitals";
const send = (m) => navigator.sendBeacon("/rum", JSON.stringify(m));
onLCP(send); onINP(send); onCLS(send);

window.addEventListener("error", (e) =>
  send({ type: "js_error", msg: e.message, src: e.filename, line: e.lineno }));
```

Tools: Sentry, OTel Browser SDK, Application Insights, Datadog RUM, LogRocket. Upload **source maps** to get readable stack traces.

---

## Privacy, security & compliance

Telemetry is **data** — it has the same obligations as your database.

- **Redact** secrets and PII at the source (logger `redact`, OTel processors)
- **Hash or tokenize** identifiers when you only need to correlate
- **Retention policies**: e.g., debug logs 7 days, audit logs 1 year
- **Access control**: not everyone needs production log access
- Regulations: **GDPR**, **CCPA**, **HIPAA**, **PCI-DSS** (never log full card numbers)
- **Audit logs** (who did what, when) are a separate, tamper-resistant concern

> OWASP Top 10: **A09 – Security Logging and Monitoring Failures** — not logging logins, failures, and access-control violations makes breaches go undetected.

---

## The tooling landscape

| Category | Open source / self-hosted | Managed / SaaS |
|---|---|---|
| Logs | Loki, Elasticsearch/OpenSearch, Vector, Fluent Bit | Datadog, Splunk, Azure Log Analytics, CloudWatch Logs |
| Metrics | Prometheus, Mimir, VictoriaMetrics | Grafana Cloud, Datadog, Azure Monitor, CloudWatch |
| Traces | Jaeger, Zipkin, Tempo | Honeycomb, New Relic, Application Insights, X-Ray |
| Dashboards | Grafana, Kibana | (built-in to most) |
| Errors / RUM | Sentry (self-host), GlitchTip | Sentry, LogRocket, Datadog RUM |
| Standard | **OpenTelemetry** | (supported by nearly all) |

**Learn the concepts + OpenTelemetry** — the vendor UI is the easy part.

---

## Cost: observability isn't free

- Most SaaS tools bill by **GB ingested**, **# of hosts**, or **# of time series**
- Logging every request body at DEBUG in prod can cost more than your servers
- Strategies:
  - Right log levels per environment
  - Sampling (traces & high-volume logs)
  - Drop/aggregate noisy data in the Collector
  - Bounded metric labels
  - Shorter retention for low-value data

---

## Putting it together: a checklist for your project

- [ ] Structured JSON logs to stdout with `LOG_LEVEL` env var
- [ ] Request ID / trace ID on every log line, returned in error responses
- [ ] Central error handler; no secrets in logs
- [ ] `/healthz` and `/readyz` endpoints
- [ ] RED metrics with bounded labels (`/metrics` or OTLP)
- [ ] OpenTelemetry auto-instrumentation for traces
- [ ] One dashboard: rate, errors, p95 latency, deploy markers
- [ ] One SLO + one symptom-based alert with a runbook
- [ ] Frontend error + Web Vitals reporting

---

## In-class exercise (10–15 min, pairs)

Scenario: **"BookNook"** — an Express API + React frontend + Postgres + external shipping-rate API.

Users report: *"Sometimes checkout hangs for ~10 seconds then fails."*

1. List **3 log events** you'd add (with fields) to diagnose this
2. Define **2 metrics** (name, type, labels) — watch your cardinality!
3. Sketch the expected **trace** for `POST /checkout` (spans + parents)
4. Write **1 SLI + SLO** and **1 alert rule** in plain English
5. Name one thing you must **never** log in this flow

<!--
Debrief (5 min): likely root cause is the shipping-rate API timing out with no client timeout. Good answers: log external call duration/outcome; histogram shipping_api_duration_seconds{outcome}; span for shipping.getRates; SLI = % checkouts succeeding < 2s; alert on error-rate or SLO burn; never log card numbers/addresses in full.
-->

---

## Key takeaways

1. **Logs** explain, **metrics** detect, **traces** locate — use all three
2. Logs should be **structured**, **leveled**, **correlated**, and **PII-free**
3. Measure **RED / Golden Signals**; use **percentiles**, not averages
4. Keep metric labels **bounded** — cardinality is a real cost
5. **OpenTelemetry** is the standard — instrument once, export anywhere
6. Alert on **user-facing symptoms** tied to **SLOs**, with runbooks
7. Observability is a **design concern**, not an afterthought

---

## Further reading

- *Site Reliability Engineering* (Google) — ch. 4 "SLOs", ch. 6 "Monitoring Distributed Systems" — sre.google/books
- *Observability Engineering* — Majors, Fong-Jones & Miranda (O'Reilly)
- OpenTelemetry docs — opentelemetry.io/docs
- The Twelve-Factor App: Logs — 12factor.net/logs
- Prometheus docs & PromQL basics — prometheus.io/docs
- W3C Trace Context — w3.org/TR/trace-context
- OWASP Logging Cheat Sheet — cheatsheetseries.owasp.org
- web.dev — Core Web Vitals

---

<!-- _class: lead -->

# Questions?

**Homework / lab idea:** add pino logging, Prometheus metrics, and OpenTelemetry tracing to your course project; view traces in Jaeger and submit a screenshot of a trace waterfall + one dashboard.
