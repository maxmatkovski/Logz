# Interview Prep — Logz.io CSE

---

## Part 1 — Home Task Review (first 15 mins)

### The story you tell in 60 seconds

> "I deployed the OpenTelemetry Demo — a realistic 27-service microservices app — on a local Kubernetes cluster using minikube. I then deployed the logz.io monitoring Helm chart to collect logs, metrics, and traces. Logs and metrics are confirmed arriving in logz.io. For traces, I have them working in the internal Jaeger and deployed the logz.io APM collector — the last step would be confirming the tracing token maps to the correct sub-account. I hit two real bugs along the way and fixed both."

---

### Questions they will ask about the lab

**"Walk me through what you deployed."**
- minikube cluster (local Kubernetes on Mac via Colima/Docker)
- OTel Demo via Helm — 27 microservices, fake e-commerce store, built-in load generator
- logzio-monitoring Helm chart — deploys 3 collectors: logs (filelog DaemonSet), metrics (OTel + Prometheus), APM (traces)

**"How are logs getting to logz.io?"**
- Every container writes to stdout → Kubernetes saves to `/var/log/containers/*.log` on the node
- logzio-logs-collector DaemonSet tails those files using a filelog receiver
- Ships via the logzio OTel exporter over HTTPS
- Zero code changes needed in the app

**"What is OpenTelemetry?"**
- Vendor-neutral open standard for logs, metrics, and traces
- Defines how apps emit telemetry + how collectors process/forward it
- logz.io is built on top of it — so data from any OTel-instrumented app can go straight to logz.io

**"What is a DaemonSet?"**
- A Kubernetes resource that ensures one pod runs on every node
- Perfect for log/metric collection — guarantees coverage regardless of cluster size
- Scales automatically when you add nodes

**"What issues did you hit?"**

*Issue 1 — Port conflict:*
- The logzio collector DaemonSet tried to bind to the same host ports as the OTel Demo's internal collector (4317, 4318, etc.) on the single minikube node
- Diagnosed via: pod stuck in `Pending`, `kubectl describe pod` showed "didn't have free ports"
- Fixed by setting `hostPort: 0` on all receiver ports in the logzio values — it only needs to collect, not receive

*Issue 2 — Helm chart secret placeholders:*
- The logzio-monitoring chart v7.x creates secrets on install but puts placeholder values (`token`, `my_env`) instead of real values
- Diagnosed via: `HTTP 401` in collector logs → decoded the secret → found placeholder
- Fixed via `kubectl patch secret` to inject the real tokens, then `kubectl rollout restart daemonset`

**"Why is APM not showing data?"**
- The logzio-apm-collector is deployed and running
- The OTel Demo collector was reconfigured to forward traces to it alongside the internal Jaeger
- The APM dashboard is still showing "Welcome" — next step: verify the tracing token is mapped to the correct logz.io tracing sub-account (logz.io has separate sub-accounts per signal type) and check `kubectl logs` on the APM collector for any auth errors

---

## Part 2 — Discovery Call Simulation (30 mins)

### The scenario
- You are a CSE at logz.io
- Discovery call with a **US-based digital gaming company**
- Meeting: **VP Engineering + Senior DevOps Manager**
- They use: **in-house ELK** for logs, **Datadog** for metrics
- Goal: understand their pain points, pitch logz.io

---

### Your discovery questions

**Opening (set the agenda):**
> "Thanks for joining. I'd love to spend the first half understanding your current setup and what's driving this conversation, then show you how logz.io could fit. Does that work?"

**Understand the current stack:**
- "How long have you been running your own ELK stack? Who owns it day to day?"
- "How many engineers are spending time maintaining it vs. shipping product?"
- "What does your Kubernetes footprint look like — EKS, GKE, self-managed?"
- "How many services are you running? Microservices or monolith?"
- "What kind of log volumes are you dealing with — ballpark GB/day?"

**Find the pain:**
- "What made you want to have this conversation today — was there a specific incident or is this more of a strategic review?"
- "What's the most frustrating thing about your current ELK setup?"
- "When something goes wrong at 2am, how long does it typically take to find the root cause?"
- "How are logs and metrics connected today — can you correlate a log to a metric spike in one place, or is it two separate tools?"
- "What's your current Datadog spend like? Is cost a factor in this evaluation?"

**Gaming-specific angles:**
- "Gaming has big spiky traffic patterns — launches, events, weekends. How does your infrastructure handle that?"
- "Do you have real-time monitoring on player-facing services? What's your SLA on incident detection?"
- "Are you doing any tracing today? If a player reports lag, how do you pinpoint which service caused it?"

**Understand the decision:**
- "Who else is involved in evaluating this — is it just engineering or is there a procurement/security side?"
- "What does success look like for you in 6 months if you made a change?"
- "What would make you NOT move forward with something like this?"

---

### Your pitch (keep it short — 5 mins max)

**The one-line pitch:**
> "logz.io is an open source-based observability platform — we give you the familiarity of ELK and Prometheus but fully managed, with logs, metrics, and traces unified in one place."

**The three points that land for this customer:**

1. **Kills the ELK maintenance burden**
   - They're running ELK in-house — someone owns that. Upgrades, scaling, disk space, index management. That's engineering time not spent on their game.
   - logz.io is managed ELK (OpenSearch) — same Kibana-like UI they already know, zero ops.

2. **Replaces Datadog at lower cost, with more context**
   - Datadog is expensive and siloed from their logs
   - logz.io puts logs, metrics, and traces in one platform — when you see a metric spike, you click through to the logs from that exact time window

3. **OTel-native — no vendor lock-in**
   - Everything we showed today uses OpenTelemetry — the industry standard
   - If they ever want to move, their instrumentation stays the same

**The workflow demo (use your lab):**
> "Let me show you quickly — this is the kind of app your team might run. [Show logz.io logs dashboard] Every service's logs are here, tagged with Kubernetes metadata — pod name, namespace, node. [Filter to a service] I can drill into just the checkout service in one click. [Show Jaeger waterfall] And here's a distributed trace — a single user action that touched 6 services. This is what incident investigation looks like instead of SSHing into servers."

---

### Objection handling

**"We already know ELK, why would we change?"**
> "You'd keep the same experience — logz.io runs OpenSearch, same query language, same Kibana-style interface. What changes is you stop managing the infrastructure behind it."

**"Datadog is already working for us."**
> "The question is whether it's worth paying Datadog prices for metrics when you could have metrics, logs, and traces unified for less. What's your current Datadog spend roughly?"

**"We're worried about vendor lock-in."**
> "That's exactly why we built on OpenTelemetry. Your instrumentation is vendor-neutral. If you ever leave logz.io, you point your OTel collectors somewhere else — nothing in your app changes."

**"Security/compliance — our logs contain sensitive player data."**
> "Good question — logz.io is SOC2 Type II certified, GDPR compliant, and offers data residency options. We can also talk about field-level redaction before data leaves your cluster."

**"This is too complex to migrate."**
> "The Helm chart we saw today is literally a one-line install. For a Kubernetes shop you're looking at a day of work for a pilot, not a migration project."

---

### Key things to remember

- **You are not expected to know logz.io inside out.** They said this explicitly. Focus on listening and asking good questions.
- **The gaming context matters.** Spiky traffic, real-time player experience, latency sensitivity — use these.
- **The 30 mins structure:** 10 mins discovery questions → 5 mins pitch → 10 mins live demo in logz.io → 5 mins next steps
- **End with a next step:** "Would it make sense to set up a technical pilot with your DevOps team for two weeks?"
