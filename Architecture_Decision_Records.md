# Architecture Decision Records — Unified Service Platform

This file holds the full decision record for each of the three foundational architectural decisions referenced in the SDD (§3) and the EA Contribution Report (§3). Each ADR captures context, the decision itself, rationale and alternatives, how it fits the architecture, a phased implementation plan, a risk register, measurable success criteria, next steps, and sign-off.

A fourth foundational decision — prioritizing and setting targets for the 10 Quality Drivers — stays documented directly in SDD §3 (Quality Attributes) rather than as an ADR here, since it's a target-setting exercise across ten drivers rather than a single decision with alternatives that were rejected.

**Status key:** *Accepted (Design)* means the decision is final for the SDD; *ARB Sign-off Pending* means formal governance review (TOGAF Phase G) has not yet run, consistent with the platform's current status (see EA_Contribution_Report.md).

---

## ADR-001: Adopt Microservices Architecture for the Unified Service Platform

**Date:** April 2025
**Status:** Accepted (Design) — ARB Sign-off Pending
**Deciders:** Enterprise Architect, Engineering Leadership, Business Sponsor

### Context

**Current State:** TechCorp Holdings runs its IT service desk on a vendor-supplied platform that deploys ticketing, change management, and problem management as a single monolithic unit.

**The Problem:**
- The vendor platform has a hard scalability ceiling — growth beyond current ticket volume risks degraded performance, with no independent path to scale just the busy parts of the system.
- Licensing fees are denominated in foreign currency, exposing the organization to FX-driven cost swings it cannot control or plan around.
- No feature ships without vendor roadmap alignment; internal teams cannot extend or customize without going through the vendor.
- All 13 service-desk functions deploy and fail together — a defect in one module puts the whole platform at risk.

**Why Now:** Cost Optimization and Centralized Service Delivery are both rated High-priority business drivers (SDD §2A), and the Budget Cap constraint (SDD §2D) makes continued FX-exposed vendor licensing unsustainable at the next renewal.

**Impact:** Engineering gains the ability to scale and deploy services independently. Operations takes on more moving parts to monitor. The business gets cost predictability and stops paying a vendor tax for features it can now build itself.

### Decision

**One-sentence statement:** TechCorp Holdings will decompose the Unified Service Platform into 8 independently deployable microservices behind a single API Gateway.

**In Scope:**
- The 8 services defined in SDD §3 (Service Desk Management, Knowledge Management, Change Management, Problem Management, Release Management, Administrative Access Management, Post-Implementation Review, Asset & Configuration Management)
- The API Gateway pattern as the single entry point (SDD §4.2)
- Kafka-based asynchronous integration between services (SDD §4.1, §4.3)

**Out of Scope:**
- Which specific technology each service runs on — covered in ADR-002
- Where the platform is deployed — covered in ADR-003
- Migration of existing vendor-held ticket/change data — belongs to the Phase F migration plan, not this decision

### Rationale

**Reason 1 — Independent scaling.** Service Desk Management can scale 5x during an incident surge without scaling Post-Implementation Review, which sees a fraction of the traffic. A monolith scales (or doesn't) as one unit.

**Reason 2 — Fault isolation.** A defect in Problem Management's root-cause workflows shouldn't be able to take down ticket creation. Microservices contain failures to the service where they originate.

**Reason 3 — Deployment agility.** Eight small teams can ship independently instead of coordinating one release train for 13 bundled functions.

**Reason 4 — Technology fit per service.** Different services have different load shapes (see SDD §4.1's scaling-property column); microservices let the platform pick the right technology per service rather than forcing one runtime to fit every workload.

**Alternatives Considered:**
- **Monolithic architecture:** rejected — the project is expected to grow, and a single deployable unit becomes harder to scale, test, and release safely as it grows. This is the same limitation the current vendor platform already has.
- **Layered architecture:** rejected — layers reduce coupling within a single deployable, but don't offer independent deployability or independent scaling, which are exactly the properties driving this decision.

### Architecture

See SDD §5.1 (System Context) and §5.2 (Container View) for the full C4 diagrams. In short: a Web Portal talks to one API Gateway, which routes to the 8 services; each service owns its own PostgreSQL schema (SDD §4.3), two use Redis for caching/sessions, and all 8 publish and consume events on a shared Kafka bus for cross-service integration.

### Implementation Phases

**Phase 1 — Platform Foundation** (Weeks 1–4)
Objective: Stand up the API Gateway and shared infrastructure (Kafka cluster, PostgreSQL cluster, Redis cluster) on OpenShift.
Success: Gateway routes to a stub service; Kafka cluster passes a producer/consumer smoke test.

**Phase 2 — Core Service Desk Migration** (Weeks 5–12)
Objective: Build and cut over Service Desk Management and Knowledge Management first — the two highest-driver-priority services.
Success: A pilot group's tickets flow entirely through the new services with no vendor-platform dependency.

**Phase 3 — Remaining Service Rollout** (Weeks 13–24)
Objective: Build and cut over Change, Problem, Release, Administrative Access, Post-Implementation Review, and Asset & Configuration services.
Success: All 8 services live in production; vendor platform reduced to read-only fallback.

**Phase 4 — Vendor Decommission** (Weeks 25–28)
Objective: Retire the vendor contract once the new platform is proven.
Success: 100% of traffic on the new platform for 30 consecutive days; vendor contract terminated.

### Risk & Mitigation

| Risk | Mitigation | Owner | Likelihood | Impact |
|---|---|---|---|---|
| Distributed debugging slows incident response during cutover | Centralized logging (EFK) and tracing from day one, not bolted on later | DevOps | Medium | Medium |
| Team unfamiliar with service-discovery and inter-service failure modes | Phase 1 foundation work is deliberately infrastructure-only, giving the team a low-stakes environment to learn the platform before Phase 2's user-facing cutover | Tech Lead | Medium | Medium |
| Data inconsistency between services during phased cutover (some tickets in vendor system, some in new platform) | Phase 2–3 run in parallel with the vendor system, not a hard cutover; reconciliation job checks for orphaned records daily during transition | Engineering | Medium | High |
| Increased operational complexity vs. the current single-vendor support model | OpenShift-native monitoring/auto-healing (SDD §5.3) absorbs most of the day-to-day burden; on-call rotation covers the rest | DevOps | High | Low |

### Success Criteria

- **Scalability:** Service Desk Management sustains a 5x traffic spike (incident-surge scenario) without breaching the 200ms response target (QD3)
- **Independent deployment:** at least one service ships a production change without requiring a coordinated release of any other service, within 3 months of Phase 3 completion
- **Fault isolation:** a simulated Problem Management outage does not degrade Service Desk Management's availability in a game-day exercise
- **Availability:** 99.9% uptime (QD2) sustained for 90 consecutive days post-cutover
- **Vendor exit:** vendor contract terminated within 28 weeks of Phase 1 start

### Next Steps

- Convene the architecture review board to move this ADR from Accepted (Design) to fully signed off — this is the Phase G governance step not yet started
- Provision the OpenShift namespace and shared infrastructure for Phase 1
- Confirm pilot group for Phase 2 with the Business Sponsor
- Stand up the reconciliation job design for the parallel-run period in Phase 2–3

### Approvals & Sign-Off

| Role | Name | Date | Status |
|---|---|---|---|
| Enterprise Architect | Chukwuemeka Nwoke | April 2025 | Signed |
| Tech Lead | [Name] | — | Pending |
| Business Sponsor | [Name] | — | Pending |
| Security & Compliance | [Name] | — | Pending |

---

## ADR-002: Select the Technology Stack

**Date:** April 2025
**Status:** Accepted (Design) — ARB Sign-off Pending
**Deciders:** Enterprise Architect, Engineering Leadership

### Context

**Current State:** With ADR-001 committing to microservices, the platform needs a concrete backend, data, messaging, and runtime stack. No comparable internal stack standard exists today — the organization currently runs the vendor's SaaS platform, not a comparable internally-owned system.

**The Problem:**
- Whatever stack is chosen must run on Linux containers on private-cloud OpenShift (a hard constraint from ADR-003), ruling out anything with a Windows-only runtime story.
- The Budget Cap constraint (SDD §2D) rules out additional per-seat or per-core commercial licensing beyond what's already committed.
- The chosen stack needs to support the Security quality driver (QD4) out of the box — OAuth2/OIDC, RBAC — rather than requiring it bolted on.

**Why Now:** This decision blocks all Phase 2+ implementation work in ADR-001; nothing can be built until the stack is fixed.

**Impact:** Directly shapes hiring/upskilling needs (SDD §3, Skillset) and the licensing cost model referenced in the platform's cost-optimization business driver.

### Decision

**One-sentence statement:** Standardize microservice backends on .NET Core, use Node.js/Express.js for the API Gateway, PostgreSQL for transactional data, Redis for caching and sessions, Kafka for asynchronous messaging, all running on OpenShift.

**In Scope:** All 8 microservices and the API Gateway (SDD §4.1, §4.2).

**Out of Scope:** Specific database schema design per service (owned by each service team); CI/CD tooling choice between Jenkins and GitLab CI (deferred to the DevOps team during Phase F — either is acceptable under this decision).

### Rationale

**Reason 1 — Enterprise maturity and security fit.** ASP.NET Core has built-in OAuth2/OpenID Connect and RBAC support, which maps directly onto the Security quality driver (QD4) without custom auth plumbing.

**Reason 2 — Cost.** Every component in this stack is open-source and free of per-core licensing on Linux — .NET Core included, as long as it runs on Linux containers rather than Windows. This directly serves the Cost Optimization business driver.

**Reason 3 — Fit for purpose per workload.** Node.js/Express's non-blocking I/O suits the gateway's job (routing many concurrent connections); PostgreSQL's ACID guarantees suit transactional ticket/change data; Redis's sub-millisecond reads suit the read-heavy Knowledge Management workload identified in SDD §6.2; Kafka's durability suits cross-service events that must not be lost.

**Reason 4 — OpenShift compatibility.** Every component containerizes cleanly on Linux, which ADR-003's private-cloud OpenShift decision requires.

**Alternatives Considered:** The original design work did not separately document rejected alternatives for individual stack components (e.g., Java/Spring instead of .NET Core, MongoDB instead of PostgreSQL, RabbitMQ instead of Kafka) — only the overall architecture style (ADR-001) and deployment target (ADR-003) went through a documented alternatives comparison. This is a real gap relative to full ADR rigor, noted here rather than backfilled with an alternatives evaluation that didn't actually happen. If this ADR goes to ARB review, closing this gap — even a lightweight comparison — should be a review condition.

### Architecture

See SDD §4.1 for the per-service technology and data-store mapping, and §5.2 for how the stack fits together in the container view.

### Implementation Phases

Follows the same phased rollout defined in ADR-001 — there is no separate phasing specific to the technology stack itself, since each phase already builds on this stack from the start.

### Risk & Mitigation

| Risk | Mitigation | Owner | Likelihood | Impact |
|---|---|---|---|---|
| Team skill gap across ASP.NET, Node.js, Kafka, and Kubernetes simultaneously | Phased rollout (ADR-001) gives the team Kafka/OpenShift exposure in Phase 1 before user-facing work in Phase 2 | Tech Lead | Medium | Medium |
| No vendor SLA on open-source components (Kafka, PostgreSQL, Redis) | Community support is mature for all four; internal platform team capacity is budgeted to absorb patching and upgrades | Engineering | Low | Medium |
| Accidental Windows deployment reintroduces the licensing cost this decision was meant to avoid | CI/CD pipeline enforces Linux-container-only build targets; documented as a hard constraint in the Technology Standards Catalogue | DevOps | Low | High |
| Kafka operational complexity underestimated | Partition sizing has not yet been load-tested against the 1000 req/sec target (SDD §6.4, still open) — this ADR's Kafka risk is directly tied to that open item | Platform | Medium | Medium |

### Success Criteria

- 80% test coverage across critical modules (QD7), measured in the CI/CD pipeline
- Redis-backed reads sustain sub-millisecond latency under the Knowledge Management read load identified in SDD §6.2
- Kafka sustains the 1000 req/sec throughput target (QD3) under load test — currently unverified, tracked as an open item in SDD §6.4
- Zero critical vulnerabilities in production, verified by the quarterly penetration test cycle (QD4)

### Next Steps

- Close the open Kafka partition-sizing item (SDD §6.4) with an actual load test before Phase 2 sign-off
- Stand up a shared library/package for cross-service standards (auth middleware, logging format) so all 8 .NET Core services don't reinvent it independently
- Confirm OpenShift base image standards with the DevOps team

### Approvals & Sign-Off

| Role | Name | Date | Status |
|---|---|---|---|
| Enterprise Architect | Chukwuemeka Nwoke | April 2025 | Signed |
| Tech Lead | [Name] | — | Pending |
| Business Sponsor | [Name] | — | Pending |

---

## ADR-003: Deploy on Private Cloud OpenShift

**Date:** April 2025
**Status:** Accepted (Design) — ARB Sign-off Pending
**Deciders:** Enterprise Architect, DevOps Leadership, Security & Compliance

### Context

**Current State:** The organization already operates a private-cloud infrastructure investment; the current vendor ITSM platform, by contrast, runs off-site in the vendor's own hosting.

**The Problem:**
- The On-Premise/Local Hosting Mandate constraint (SDD §2D, High priority) requires the platform to run on infrastructure the organization controls — no public cloud.
- Data Residency Compliance (NDPR) requires user and ticket data to stay within approved local jurisdictions.
- Any public-cloud spend would reintroduce the same FX exposure the platform is being built to eliminate (SDD §2A, Cost Optimization).

**Why Now:** This is a hard constraint, not a preference — it must be settled before any infrastructure provisioning in ADR-001's Phase 1 can begin.

**Impact:** The infrastructure team owns capacity planning directly, rather than relying on public-cloud elasticity to absorb demand spikes.

### Decision

**One-sentence statement:** Deploy the Unified Service Platform exclusively on the organization's private-cloud OpenShift environment; no production workload or data leaves organization-controlled infrastructure.

**In Scope:** Production and disaster-recovery environments; self-hosted CI/CD runners (Jenkins or GitLab CI).

**Out of Scope:** Developer sandbox/local environments (may use local containers on developer machines — not a compliance concern); future hybrid-burst capacity to public cloud is not decided against permanently, just out of scope for this decision.

### Rationale

**Reason 1 — Compliance is non-negotiable.** The on-premise mandate and NDPR data-residency requirement aren't cost-optimization choices — they're constraints the architecture has to satisfy regardless of what a public-cloud comparison would otherwise favor.

**Reason 2 — Cost predictability.** A capex model against existing private-cloud infrastructure avoids the unpredictable, FX-exposed opex of public-cloud consumption billing.

**Reason 3 — Automated deployment still works.** CI/CD pipelines (Jenkins/GitLab CI, Terraform/Ansible for infrastructure-as-code) deliver the same automated, repeatable release process a public-cloud deployment would offer — private hosting doesn't mean manual operations.

**Alternatives Considered:**
- **Public Cloud (Azure):** rejected — violates the on-premise mandate and reintroduces FX exposure through cloud billing.
- **Hybrid Cloud:** rejected as unnecessary — current and projected capacity fits within the private cloud's existing footprint; adding a hybrid split would add operational complexity with no corresponding benefit given the compliance constraint already rules out using public cloud for any regulated data.

### Architecture

See SDD §5.3 for the full OpenShift deployment topology — private load balancer, per-service deployments with independent Horizontal Pod Autoscalers, shared Kafka/Redis/PostgreSQL clusters, and the monitoring/logging stack (Prometheus/Grafana, EFK).

### Implementation Phases

**Phase 1 — Cluster Provisioning** (Weeks 1–3)
Objective: Size and provision the OpenShift cluster (VM/bare-metal worker nodes) against Phase 1 of ADR-001.
Success: Cluster passes a capacity smoke test at 2x current vendor-platform peak load.

**Phase 2 — Shared Services Standup** (Weeks 3–5)
Objective: Deploy the internal PostgreSQL cluster, Redis cluster, and Kafka cluster with backup/PITR configured.
Success: Daily full backups and hourly transaction-log backups running and verified restorable.

**Phase 3 — CI/CD Cutover** (Weeks 5–8, overlapping ADR-001 Phase 1–2)
Objective: Self-hosted Jenkins/GitLab CI runners live inside the private cloud, Terraform/Ansible managing infrastructure as code.
Success: A code change deploys to a non-production namespace with zero manual steps.

**Phase 4 — Disaster Recovery Drill** (Week 26, before vendor decommission in ADR-001 Phase 4)
Objective: Validate the RTO/RPO targets under a simulated cluster failure.
Success: Recovery completes within the <4hr RTO / <1hr RPO targets (QD10).

### Risk & Mitigation

| Risk | Mitigation | Owner | Likelihood | Impact |
|---|---|---|---|---|
| No public-cloud elasticity to burst to if demand exceeds private-cloud capacity | Capacity planning sized to 10x growth target (QD1) up front, with headroom audited quarterly | Infrastructure | Medium | High |
| Hardware procurement lead time delays scaling beyond planned capacity | Procurement lead times factored into the capacity plan with a 6-month buffer ahead of projected growth | Infrastructure | Medium | Medium |
| Single-region hosting — no geographic redundancy the way multi-region public cloud offers | Multi-cluster hosting within the private environment (SDD §3, Deployment Strategy) for failover; accepted residual risk is single-site outage, mitigated by the DR drill in Phase 4 | DevOps | Low | High |
| DR drill reveals RTO/RPO targets aren't actually met | Drill scheduled before vendor decommission (ADR-001 Phase 4), so there's a fallback if targets aren't hit on the first attempt | DevOps | Medium | High |

### Success Criteria

- RTO < 4 hours and RPO < 1 hour (QD10), validated by an actual DR drill, not just a design target
- 99.9% uptime (QD2) sustained over 90 consecutive days post-cutover
- Zero data residency violations — 100% of production data confirmed hosted within approved jurisdictions at the Phase 2 audit
- Infrastructure cost variance stays within the Budget Cap constraint's approved range through the first year of operation

### Next Steps

- Finalize node sizing with the infrastructure team against the 10x growth target
- Confirm OpenShift support contract renewal date falls outside the Phase 1–4 rollout window
- Schedule the first DR drill for Phase 4
- Security & Compliance review of network segmentation before Phase 1 provisioning begins

### Approvals & Sign-Off

| Role | Name | Date | Status |
|---|---|---|---|
| Enterprise Architect | Chukwuemeka Nwoke | April 2025 | Signed |
| DevOps Lead | [Name] | — | Pending |
| Security & Compliance | [Name] | — | Pending |
| Business Sponsor | [Name] | — | Pending |
