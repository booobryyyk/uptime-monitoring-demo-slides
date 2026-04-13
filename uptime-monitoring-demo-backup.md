# Uptime monitoring

# 0. Agenda

1. Objective
2. Industry standards
3. Monitoring state
4. Architecture
5. Live demo of adding a new page to monitoring
6. Key takeaways

# 1. Objective

- Higher delivery speed, frequent changes = higher risk of something breaking unnoticed
- Detection time was random
- Existing monitoring is pricy and didn’t cover all pages across all domains
- Single incident may have a potential loss of ~$5k

# 2. Industry standards

The “Nines” of Availability:

| Availability | Downtime/year | Downtime/month | Typical use case |
| --- | --- | --- | --- |
| 99% | 3.65 days | 7.31 hours | Internal tools |
| 99.9% | 8.77 hours | 43.8 min | Most SaaS / web apps |
| 99.95% | 4.38 hours | 21.9 min | Cloud providers (e.g. AWS EC2 SLA) |
| 99.99% | 52.6 min | 4.38 min | Payment systems, e-commerce |
| 99.999% | 5.26 min | 25.9 sec | Telecom, healthcare, finance |

Golden standard: 99.9%-99.95% for commercial web apps. Five nines (99.999%) is the theoretical ideal, but even Google says it’s not attainable in practice.

Key facts (Uptime Institute, 2024-2025):
- 54% of significant outages cost over $100,000; 1 in 5 exceeded $1 million
- 40% of major outages caused by human error

Monitoring techniques:
- Synthetic Monitoring — simulated requests at regular intervals from multiple locations. Proactive — catches outages before users report them.
- Real User Monitoring (RUM) — JavaScript on pages measures actual user experience. Reactive — shows real-world impact.
- Error Budget / SLO-based Alerting (Google SRE approach) — define an SLO (e.g. 99.9%), track an error budget (allowed downtime per quarter), alert when the budget burn rate is too high.
- Chaos Engineering (Netflix approach) — deliberately inject failures (Chaos Monkey, Latency Monkey) to
verify resilience. Netflix built an entire "Simian Army" for this.

Best practice is to combine both: synthetics for proactive detection, RUM for real-world truth.

# 3. Monitoring state

- We’re using self-hosted solution Uptime Kuma (https://status.mate.academy). Currently, we’re using it as a status checker that pings pages with a fixed interval - 1 minute for pages with higher traffic and 5 minutes for ones with lower (blog pages)
- We have a dedicated db that stores history for heartbeats
- We detected an incident with memory issues on frontend landings. Monitoring was the catalyst that surfaced the issue early — it could have hit us under higher traffic at a worse time
- Cross product monitoring. Kodree already moved their monitoring from Sentry to Uptime Kuma

# 4. Architecture

Self-hosted Uptime Kuma backed by MariaDB on AWS RDS

1. Uptime Kuma is a self-hosted monitoring tool. Here's its architecture:
    
    Core Architecture:
    
    - Single-process Node.js monolith with WebSocket-first design. Single-threaded loop with setInterval:
  Each monitor runs as a separate setInterval(beat, interval * 1000). There's no worker pool or thread pool — all monitors share the single Node.js event loop. The beat() function does the actual check (HTTP
  request, TCP connect, ping, etc.), records the result to the DB, and emits the status to all connected WebSocket clients via Socket.IO.

  Why this works (and where it breaks):
  - HTTP checks are async I/O — Node handles thousands of these concurrently on one thread without blocking, since they're just waiting on network responses
  - The event loop only blocks on CPU-bound work, which health checks aren't — they're almost entirely I/O-bound (send request, wait for response)
  - The real bottleneck is DB writes — every heartbeat creates a row, so with many monitors at short intervals, SQLite/MariaDB write throughput becomes the limit, not Node

  Practical implication for your demo:
  This is actually a good talking point — Uptime Kuma can comfortably handle hundreds of monitors on a single Node process precisely because of how Node's event loop works with async I/O. It's a strength of the
  architecture for this use case, not a limitation.
    - Monitor engine - a loop-based scheduler where each monitor runs on its own interval (e.g., every 60s). Monitors execute checks (HTTP, TCP, ping, DNS,
    Docker, etc.) and emit status via Socket.IO
    - Notification channels — 90+ integrations
2. Backed by MariaDB (open-sourced fork from MySQL but mostly more performant than MySQL)
3. Code-first approach: Knex seeders as migrations (no manual UI config, but still available)
4. Kubernetes (ArgoCD), Helm chart, SealedSecrets for credentials
5. Why code-first matters: reproducibility, review process, version control

# 5. Live

[request adding a new monitoring for page through Slack]

@Claude, please create a PR with a seeder in uptime-monitoring to add https://mate.academy/lpa/marathon/data-analyst-marathon with a 5-minute interval

# 6. Key takeaways

Conclusions:
- Infrastructure-as-code for monitoring = fewer manual errors, full review process
- Adding a new page to monitoring is a single seeder file
- Idempotent operations — safe to re-run, easy to rollback

Lessons learned:
- Start with synthetic monitoring first — it covers the most ground with least effort
- Code-first approach pays off quickly: no config drift, easy onboarding, reviewable changes
- Monitoring exposed issues we didn't know we had (frontend landings memory leak)
- Cross-team adoption (Kodree) validated that the solution scales beyond one team

Next steps:
- Coverage expansion across all domains
- New notification channels
- Response time monitoring