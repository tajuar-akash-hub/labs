# R&D : Labs 1, 2, and 3: Alternatives, Trade-offs, and Production Readiness

This document goes one level deeper than the POCs. For each lab, it looks at: what other approaches could have been used, why we picked what we picked, what it costs (in money and complexity), and what would still need to be done before this could run in real production.

---

## Lab 1 : Private ML Inference (ViT + Transit Gateway)

### 1. Alternatives Considered

| Approach | How it works | Why we used / didn't use it |
|---|---|---|
| **Transit Gateway (used)** | Central hub connecting multiple VPCs | Scales to many VPCs cleanly; one place to manage routing; supports route tables per attachment for isolation |
| **VPC Peering** | Direct 1-to-1 connection between two VPCs | Simpler and cheaper for exactly 2 VPCs, but does **not** scale — with 5+ VPCs you'd need a full mesh of peering connections, which becomes hard to manage. No route-table-per-attachment control either. |
| **PrivateLink (VPC Endpoint Service)** | Exposes a specific service (not the whole VPC) across VPC boundaries | Better if you only want to expose *one* API, not general network reachability. Would actually be a strong alternative here if the client only ever needs the `/predict` endpoint and nothing else — worth considering as a tighter-scoped option. |
| **Public endpoint + strict auth (API Gateway + IAM/Cognito)** | Model is technically internet-facing but locked down with authentication | Rejected for this use case — the requirement was "not reachable from the internet at all," not "reachable but authenticated." Auth can be misconfigured; network isolation cannot be bypassed by a config mistake in the app layer. |

### 2. Trade-offs of the Chosen Approach (Transit Gateway)

- **Pro:** Centralized routing control, scales to many VPCs, supports fine-grained route tables so you can isolate groups of VPCs from each other (as shown in Lab 4).
- **Con:** TGW has an hourly cost per attachment plus a per-GB data processing charge — this adds up if there is a lot of traffic between VPCs, more than VPC Peering (which has no hourly charge, only data transfer cost).
- **Con:** Slightly more setup complexity than peering — route tables, attachments, and propagation settings all need to be understood.

**When to prefer PrivateLink instead:** if the "client" is actually many different teams/accounts that should only call one specific model API and should never have any other network reachability into the model VPC, PrivateLink is a tighter blast radius than TGW. TGW gives network-level reachability; PrivateLink gives service-level reachability only.

### 3. Production Readiness Gaps

The POC proves isolation and basic function. It is **not** production-ready as-is. Missing pieces:

- **No load balancing / no autoscaling** — a single EC2 instance is a single point of failure. Production would need the model server behind a Network Load Balancer with an Auto Scaling Group across multiple Availability Zones.
- **No authentication between client and model** — the POC relies entirely on network isolation. In production, even internal services should authenticate to each other (mTLS or signed requests) — this is "defense in depth," in case someone accidentally gets network access they shouldn't have.
- **No observability** — no logs, metrics, or alerting are wired up in this lab. You'd want the same Prometheus/Grafana pattern from Lab 2 attached here too.
- **No model versioning strategy** — the POC pulls one static model from S3. Production needs a plan for rolling out new model versions without downtime (blue/green or canary deployment).
- **No cost guardrails** — TGW data processing charges can surprise you at scale; would need monitoring/budgets set up before heavy production traffic.
- **Manual instance management** — the POC launches a raw EC2 instance. Production should use something like ECS, EKS, or an Auto Scaling Group with a launch template, so unhealthy instances are replaced automatically.

### 4. Recommendation

Keep Transit Gateway if you expect more than 2-3 VPCs needing access over time (matches Lab 4's registry pattern). Add a load balancer and auto-scaling before this touches real traffic. Consider PrivateLink specifically for the client-to-model path if you want to narrow the blast radius further than "reachable VPC," down to "reachable API only."

---

## Lab 2 : Multi-Region Failover (BGP + Prometheus/Grafana)

### 1. Alternatives Considered

| Approach | How it works | Why we used / didn't use it |
|---|---|---|
| **BGP route withdrawal (used)** | A route is removed from the routing table the moment a health check fails | Fastest possible failover — reconvergence in seconds, not tied to any cache or TTL |
| **Route 53 DNS Failover** | DNS answers change based on health checks | Much simpler to set up (a few clicks), but failover speed depends on DNS TTL and resolver/client caching — can take anywhere from tens of seconds to several minutes in the real world, since not all clients respect TTL properly |
| **AWS Global Accelerator** | Anycast IP with automatic health-check-based routing to the nearest healthy region | A strong AWS-native alternative to hand-rolled BGP — much less operational overhead than running FRRouting yourself, and still fails over in seconds. Doesn't require you to manage BGP sessions at all. |
| **Application-level retry/failover** | The client itself tries Region A, and on failure, retries Region B | Simple, no networking changes needed, but pushes complexity into every client application, and doesn't help if the client itself can't tell the difference between "slow" and "down." |

### 2. Trade-offs of the Chosen Approach (BGP)

- **Pro:** Genuinely the fastest failover mechanism available — this is why real ISPs, CDNs, and telecoms use it.
- **Pro:** Works below the application layer, so it protects against a whole category of failures (region-wide outage) without every single client needing custom failover logic.
- **Con:** Significant operational complexity — someone on the team needs to actually understand BGP to operate and debug this. Misconfigured route-maps or a flapping session can cause outages of their own.
- **Con:** In the lab we simulated an "on-prem" BGP router using FRRouting in a container. In real production AWS, you'd want this to run over an actual **Direct Connect** connection or **AWS Site-to-Site VPN with BGP**, which has its own setup cost and, for Direct Connect, a lead time of days to weeks to provision physical circuits.

**Honest comparison:** for most companies, **AWS Global Accelerator** gets you 90% of the benefit (fast, automatic, health-check-driven failover) with a fraction of the operational burden of running your own BGP sessions. BGP is the right call specifically when you already operate hybrid/on-prem infrastructure and need routing control that spans your own network and AWS together — which is the scenario Lab 2 and Lab 5 assume.

### 3. Production Readiness Gaps

- **Health check is too simple** : the POC only checks `/health`, which just confirms the process is running. Production should check a real synthetic transaction (actually run a small transcription and confirm the result is correct) so it also catches "the process is up but the model is broken."
- **No automatic failback safety** — the lab fails back to Region A automatically once it's healthy again. In production, sudden failback under load can cause a second outage if Region A comes back up but isn't actually warmed up / caught up. Consider a manual failback approval step, or a gradual traffic shift.
- **No alerting** — Grafana shows the failover visually, but nobody is paged. Alertmanager (or an equivalent) needs to be wired to page on-call the moment a route is withdrawn.
- **Single BGP router per region is a single point of failure itself** — production BGP setups usually run two routers per site for redundancy, so the router itself isn't the next thing to fail.
- **Data consistency between regions isn't addressed** — this lab only handles compute failover. If the model also depends on a database, you need a real cross-region data strategy (replication, conflict resolution) which is a much bigger project on its own.

### 4. Recommendation

For most teams: start with AWS Global Accelerator, which solves the same problem with far less operational overhead, and only move to full BGP if you already have (or are already building) hybrid on-prem/AWS connectivity for other reasons. If you do go the BGP route, invest early in a strong health check and paging setup — the fast failover is only valuable if it's also trustworthy.

---

## Lab 3 : Secure Voice Model over IPsec

### 1. Alternatives Considered

| Approach | How it works | Why we used / didn't use it |
|---|---|---|
| **Site-to-Site IPsec VPN (used)** | Encrypts all traffic between two networks at the network layer | Good fit for a fixed "branch office" style connection (like a hospital's main office); encrypts everything automatically, app code doesn't need to know encryption is happening |
| **TLS/HTTPS only (no VPN)** | Encrypt just the application traffic, not the whole network path | Much simpler to set up (no VPN infrastructure needed), but only protects the specific HTTP connection — doesn't hide network-level metadata, and still typically requires the model endpoint to be reachable in some way from the client's network, which conflicts with "fully private" |
| **AWS Client VPN** | VPN designed for individual users/devices connecting in, rather than site-to-site | Better fit if this were individual remote workers instead of a whole office/hospital network — not the right shape for this lab's scenario |
| **PrivateLink + TLS** | Expose only the specific API privately, encrypt with TLS on top | Strong alternative if the on-prem side already has private connectivity to AWS (e.g. via Direct Connect) and you just want to add application-layer encryption on top |

### 2. Trade-offs of the Chosen Approach (IPsec VPN)

- **Pro:** Encrypts everything automatically at the network level — nobody has to remember to add TLS to every single request; it's enforced by the network path itself.
- **Pro:** This is the pattern auditors and compliance teams are most familiar with for healthcare/finance requirements, which makes the audit conversation easier.
- **Con:** Adds a piece of infrastructure (the VPN tunnel itself) that can fail independently — if the tunnel drops, everything behind it stops working, even if AWS and the on-prem network are each individually fine.
- **Con:** In this lab we used **static routing** for simplicity. Static routes require manual updates any time the on-prem network changes — this doesn't scale well beyond one fixed office network (this is exactly the gap Lab 5 closes by using BGP dynamic routing instead).
- **Con:** A single VPN tunnel is also a single point of failure — production should use both of the two tunnels AWS provisions by default, not just tunnel 1 as this lab did for clarity.

### 3. Production Readiness Gaps

- **Only one tunnel configured** — AWS gives you two tunnels for redundancy; the lab only wired up one on the on-prem side. Production must configure both, with automatic failover between them.
- **No TLS on top of the VPN** — the lab relies on IPsec alone. Best practice is "belt and suspenders": encrypt at the network layer (IPsec) **and** the application layer (HTTPS/TLS) so a single misconfiguration doesn't fully expose the traffic.
- **No monitoring of tunnel health** — if the tunnel silently drops, nobody would know until someone notices requests failing. Needs CloudWatch alarms on VPN tunnel state.
- **Static routing doesn't scale** — as noted above, this should move to BGP (Lab 5's pattern) for any real deployment.
- **No key rotation plan** — the pre-shared key generated in this lab is meant to be treated as a long-lived secret, but production should have a defined rotation schedule and a process for it that doesn't cause downtime.
- **No formal compliance mapping** — proving "traffic is encrypted" with a packet capture is a good demo, but a real compliance program (HIPAA, PCI, etc.) needs this mapped to specific control requirements and documented as part of a formal audit trail, not just a one-time lab screenshot.

### 4. Recommendation

The IPsec approach is the right one for this scenario (a fixed on-prem network with compliance requirements). Before production: configure both tunnels for redundancy, add TLS on top for defense in depth, add tunnel-health alerting, and move from static to BGP routing if the on-prem network topology is expected to change over time.

---



