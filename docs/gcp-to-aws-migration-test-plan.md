# GCP to AWS Website Migration Test Plan

## About this plan

**What it is.** A GCP-to-AWS move losing no data, functionality, security posture or performance: what is tested, when, by whom, against which threshold, when to roll back.
**Altitude.** Documentation only: no test run, infrastructure or cloud account; approach level, one sitting's read; role names only; no uninvited brand, date, person or team.
**Runbook.** Referenced, never replaced; unagreed values stay bracketed (`[Hypercare duration]`), listed at 8.2.
**Identifiers.** `P1`-`P7` phases, `TA-1`-`TA-9` areas, one meaning each; severities written out (`highest-severity`, `major-severity`), never phase ids; `[Max P1 open age]`/`[Max P2 open age]` are ages.

## 1. Strategy and approach

### 1.1 Test levels

Four levels: **smoke** — 4.1's eight checks, per traffic step, on the user-reachable site, drawn from functional and regression, integration and DNS/TLS/CDN; **functional and regression**, system level; **provider-contract integration**; **non-functional** — performance, security, resilience, observability, DNS/TLS/CDN.
Not levels: **data integrity** (TA-2), cross-environment comparison; **rollback** (TA-9), rehearsal of the whole cutover sequence.

### 1.2 The assumed cutover model

**A parallel run on the target, then a progressive weighted DNS shift.** Both serve; the source stays live and in sync — system of record, way back — to 4.3's hypercare exit; a step reverses by weight.
A step advances only once its monitoring window has closed inside every threshold and the owner has recorded sign-off; the parallel run hands over to the first step once 5.1 is green; hypercare ends on 4.3's exit signals.

| Stage | Weight | What happens | On breach |
|---|---|---|---|
| Parallel run | 0% | Source serves; target exercised directly; replication keeps the two in step | Fix, re-enter; no step starts |
| Steps 1-4 | 1%, 10%, 50%, 100% | Smoke set (4.1), `[Step monitoring window]` | Reverse the weights; source serves all |
| Hypercare | 100% to target | `[Hypercare duration]`; source live and in sync, way back available | Reverse while the way back holds |

Weighted records relative; one TTL per group; a step advances by weight change, never address edits; TTL to `[TTL value]` 24-48 hours ahead, the shift waiting out the previous TTL before the first step, each change effective inside `[TTL propagation window]`; mixed traffic per step expected and tested, never an anomaly (TA-8). Stated once here; section 4 does not restate it.

### 1.3 Test data

TA-1, TA-2 and TA-4 need an **anonymised production copy** at realistic volume and distribution; no live customer personal data in non-production; redaction or tokenisation precedes the copy; without it no area reports.

### 1.4 Source-to-target mapping

| Concern | Source (GCP) | Target (AWS) |
|---|---|---|
| Compute | Compute Engine, GKE, Cloud Run | EC2, EKS, ECS Fargate |
| Relational database | Cloud SQL | RDS, Aurora |
| Object storage | Cloud Storage | S3 |
| Load balancing and CDN | Cloud Load Balancing, Cloud CDN | ALB, CloudFront |
| DNS and certificates | Cloud DNS, Google-managed certificates | Route 53, ACM |
| Identity and secrets | IAM, Secret Manager | IAM, Secrets Manager |
| Observability | Cloud Monitoring, Cloud Logging | CloudWatch, X-Ray |
| Edge security | Cloud Armor | AWS WAF, Shield |

Undescribed stack: the mapping is tested, not re-decided; no row can be excluded outright; each an assumption (8.1), covered at 7.2. Edge security: no edge firewall today — new config, not a port; a named-owner gap.
Identity and secrets: no end-user identity provider; the row reduces to workload and platform identity and secret storage, TA-3's post-move assertions then covering internal service-to-service calls only.

Where the target's configuration model differs, not only its defaults, the configuration is rebuild work, never a port: CDN configuration, cache policies and any edge code rebuilt, re-tested against the source's observed behaviour before cutover (TA-8).

### 1.5 Programme requirements

1. **All nine test types** in scope, each a test area in section 3.
2. **A dry-run cutover in a production-like target before the real one**, smoke set passing at every simulated step (2.4).
3. **The rollback rehearsed twice, consecutively, in P4 before the first production traffic shift**, each covering post-shift writes inside `[Max rollback time]`; overrun fails P4.

### 1.6 Conventions

**Phased and gated:** no phase starts before the previous exit criteria are signed off. **Baseline-then-compare:** source first. **Rollback-first:** way back rehearsed and timed before the first shift, the source serving and receiving replicated change to the hypercare exit. **One test-area format:** objective, method, example checks, pass threshold, owner.
**Thresholds checkable (1.9):** count, percentage, latency, error rate, lag, baseline comparison or a binary gate — a signed exit criterion, a freeze in effect; never "looks fine". **Evidence:** a recorded artefact — report, dashboard capture or signed checklist line.

### 1.7 Tool categories and limits

| Category | Used by | What it verifies |
|---|---|---|
| Load generation | TA-4 | Latency, throughput, error rate under normal, peak, spike load |
| Browser and API regression | TA-1, TA-3 | Journeys, provider calls on both sides |
| Data comparison | TA-2 | Rows, checksums, aggregates, schema |
| Vulnerability scanning | TA-5 | Package, image and application findings |
| Synthetic monitoring | TA-7, TA-8 | External journey checks after each step |
| DNS and TLS inspection | TA-8 | Resolution, TTL, certificate chain, handshakes |

Category names only, never brands. Limits: load generation cannot judge data correctness, authorisation or rendering; browser and API regression cannot see infrastructure timeouts, firewall or NAT behaviour, or behaviour under load; data comparison cannot judge business rules or data-dependent behaviour; vulnerability scanning cannot prove identity-policy equivalence or reach the whole estate; synthetic monitoring cannot reach unscripted paths or anything under load; DNS and TLS inspection cannot see application behaviour or allow-list state.
No category proves identity-policy equivalence, allow-list re-registration, data reconciliation or a **failover decision**'s correctness: those stay manual checks with a named owner.

### 1.8 Scope boundaries

**In scope:** the seven phases and their gates; the nine areas with methods and thresholds; the mapping and its coverage; cutover steps and windows; hypercare; checklist, triggers and rehearsals; roles, environments and risks; traceability; assumptions.
**Out of scope:** test cases, scripts, automation; re-architecting, feature change, FinOps; the runbook; non-website workloads; target build, edge rules, allow-lists; UI, component library, design asset; infrastructure creation, cloud-account access.

### 1.9 How this plan is verified

**Walkthrough against the nine success criteria:** per phase owner, entry and exit criteria, test types; a threshold per area; measurable gates and triggers; rollback covering writes inside `[Max rollback time]`; a decision and action per failure state; every mapping row covered; two-way traceability; altitude; assumptions marked.
**Two traceability checks:** 6.4 against every area, 1.4 against the areas, no orphan (7.1, 7.2). **The threshold check:** every pass threshold, go/no-go entry and rollback trigger objectively checkable per 1.6 — none on "looks fine".
**The altitude check:** approach level for a single sitting, detail in tables; held to the roughly five-to-ten-page guide, which the mandated inventory runs past — the content kept whole rather than cut to the guide; no procedures, test cases, scripts or code; no brand, date, person or team the user did not supply.

## 2. Phase-by-phase test activities

Seven phases in order: none starts before the previous exit criteria are signed off by its owner. Types per phase: 2.8.

**2.1 P1 Pre-migration baseline** — Test Lead; types 2.8. **Entry:** source accessible; roles assigned; anonymised copy producible (1.3).
**Exit:** baselines (functional; performance normal, peak, spike); data profile (per-table row counts, checksums, object counts, hashes); inventory (identity, network, TLS, edge, secrets, encryption); dashboards, alerts and logs with thresholds; replication established, initial sync verified; journeys scripted or manual.

**2.2 P2 Target environment validation** — Cloud Platform Engineer; types 2.8. **Entry:** P1 exit signed off; production-sized target built; test access; connectivity.
**Exit:** smoke set served, no certificate warning or placeholder; every alert proven by fault injection; identity, segmentation, TLS, edge rules, secrets and encryption verified; failover rehearsed with measured RTO/RPO; backup restored; DNS verified; every gap with no source equivalent owned.

**2.3 P3 Data migration validation** — Data Engineer; types 2.8. **Entry:** P2 exit signed off; replication running; anonymised copy both sides.
**Exit:** counts, checksums, schema and object counts match outside an accepted difference list; zero orphan rows; lag inside `[Max replication lag at freeze]` for `[Stability window]`; consistency mechanism proven; every mismatch resolved, or an accepted difference with a named owner.

**2.4 P4 Pre-cutover rehearsal** — Cutover Manager; types 2.8. **Entry:** P3 exit signed off; production-like target and runbook available; dry-run window agreed.
**Exit:** dry-run complete, smoke set passing at every step; rollback rehearsed twice inside `[Max rollback time]`, each with post-shift writes; allow-list, callback and redirect URLs re-registered; go/no-go checklist walked and measurable; support rota named.

**2.5 P5 Cutover** — Cutover Manager; types 2.8. **Entry:** P4 exit signed off; every go/no-go entry green; freeze in effect; TTL lowered; weight staged at zero; rota staffed.
**Exit:** each step's smoke test passed, its window closed inside threshold, sign-off before the next step; 100% of traffic on the target; shift-window writes reconciled; triggers monitored, confirmed not breached.

**2.6 P6 Post-cutover hypercare** — Application Owner; types 2.8. **Entry:** P5 exit signed off; traffic fully shifted.
**Exit:** hypercare for `[Hypercare duration]`; no highest-severity incident beyond `[Max P1 open age]`, none major beyond `[Max P2 open age]`; latency and error in threshold; source receiving no writes; every open issue documented with an owner and a next action.

**2.7 P7 Source decommission readiness** — Cloud Platform Engineer; types 2.8. **Entry:** hypercare exit met, signed off by the Application Owner and the Cloud Platform Engineer.
**Exit:** no production traffic to the source across `[Decommission observation window]`; reconciliation proving no source-only data or write conflicts; rollback decision closed.
Source credentials, firewall rules, DNS records, monitoring and allow-list entries listed for removal, each owned, against a removal checklist, no date fixed by this plan; configuration archived for audit.

### 2.8 Test types run per phase

The single statement of which types run in which phase.

| Phase | Test types run |
|---|---|
| P1 | Functional and regression, performance, data integrity, security, observability, DNS/TLS/CDN |
| P2 | Functional and regression (smoke), security, resilience and failover, observability, DNS/TLS/CDN, integration |
| P3 | Data integrity; integration; performance at volume |
| P4 | Functional and regression, integration, rollback, DNS/TLS/CDN, observability, performance, security, resilience and failover |
| P5 | Functional and regression (smoke per step), DNS/TLS/CDN, observability, data integrity, integration, rollback |
| P6 | Functional and regression, performance, data integrity, integration, observability, resilience, DNS, TLS and CDN, rollback |
| P7 | Data integrity (final reconciliation); functional and regression; security (access removal) |

## 3. Test areas

Nine areas, one per test type, five fields each: **objective, method, example checks, pass threshold, owner**. Types per phase: 2.8, the single statement, not repeated here; a phase named under a block places that block's check, not that phase's test-type set.

| Area | Objective | Method | Example checks | Pass threshold | Owner |
|---|---|---|---|---|---|
| 3.1 TA-1 Functional and regression | Journeys, admin functions, jobs, outbound messages, authentication and authorisation as on source | Baseline replay; admin actions as administrator, a non-privileged role attempting the same; same accounts and roles both sides — sign-in, session establishment, session lifetime, session renewal, sign-out; page-level tag firing, each page's analytics request, both hostnames; jobs by run logs | Browse; login, sign-out; search; checkout; accounts; admin console; non-privileged denial; job output, run count; email; webhook signature; tag firing once per page view, expected identifiers on target hostnames, analytics request resolving there; invalid-request error page | 100% journeys at baseline; zero sign-in/session-establishment/lifetime/sign-out difference vs source; 100% role-based checks pass, non-privileged denial included; zero failed admin actions; zero duplicate or missing firings per page view; jobs once per schedule, no duplicates or missed runs; 100% emails and webhooks delivered, zero duplicate sends | Application Owner; QA / Test Engineer |
| 3.2 TA-2 Data integrity | No loss, corruption or duplication; both sides agree in the run | Counts, checksums, aggregates, schema, referential integrity, application reads; per-object hashes; lag both sides through write freeze; engine settings, parameter groups by definition | Row counts; checksums; schema objects; orphan-row child-to-parent anti-joins; object hashes; lag at freeze; settings and parameter groups compared, naming immutable-after-creation, uneditable-in-default-group and restart-required settings, table-name and identifier case sensitivity | Zero count/checksum/schema/object/hash mismatches; zero orphan rows; lag inside `[Max replication lag at freeze]` both sides for `[Stability window]`; record counts for every reconciled write equal to those written during the parallel run; zero unrecorded settings differences, all immutable-after-creation, default-group and restart-required ones named before P3 exits, matched or accepted with an owner; zero table-name/identifier casing differences outside accepted list | Data Engineer |
| 3.3 TA-3 Integration | Third parties accept target traffic; no allow-list, callback or redirect at source | Egress contract checks from target's egress addresses against each provider — APIs, payment, email, identity, analytics, tag managers; re-registration against target hostnames; response-code comparison, source side replayed from P1 baseline, never by new traffic to source production; retired source addresses tested; post-move sign-in, session lifetime, sign-out and role-based access asserted for administrative users on TA-1's checks and thresholds | Identity callback; payment; outbound email; webhook signature; analytics ingestion from target hostnames; consent state, cookie domain, cross-domain settings against them, collection endpoint's view; storage hostname a third party calls back to, certificate matching that name; retired allow-list entry; old callback URL; each provider's delivery logs | 100% endpoints reachable; zero source-address rejections; registrations verified against target hostnames; no residual allow-lists or callbacks; zero callbacks at source after final step, from providers' own delivery logs; zero consent, cookie-domain or cross-domain defects; 100% tag and collection requests resolving against target hostnames | Third-Party Integration Owner, with Cloud Platform Engineer |
| 3.4 TA-4 Performance and load | Baseline met under normal, peak, spike load; scaling observed | Identical source-then-target scenarios, same volume, journey mix, think times, durations | p50, p95, p99; error mix; throughput; scale-out; saturation; queue depth | p95 within `[Agreed latency tolerance]`; error rate at baseline plus `[Agreed error-rate tolerance]`; throughput at least baseline; scale-outs inside `[Max scale-out time]`; saturation under `[Max sustained utilisation]`; `[Max scaling events]` per spike window | Performance Engineer, with SRE / Observability Engineer |
| 3.5 TA-5 Security | Posture at least equal to source's, no new exposure | Policy review; configuration inspection; TLS handshakes; package, image, runtime scanning; identity policy walked policy by policy against its target equivalent; ported edge rules staged count-only or non-blocking before enforcement | Wildcard permissions; internet paths; TLS versions; HSTS, subject-alternative-name coverage per host; edge rule parity; secrets absent from code, config, images, readable only by intended identity; admin path, multi-factor; secret rotation; policy walk naming explicit deny precedence, organisation guardrails, permission boundaries, trust policies, additive bindings, deny policies, inheritance, impersonation, and a folder- or organisation-level grant on target; six edge attributes one by one — managed rule sets, body-inspection limits, over-long query strings, rate-based thresholds, ban windows, attachment points; count-only match records vs source's observed behaviour | Zero over-privileged production grants; no unintended internet-reachable port; TLS 1.0 and 1.1 rejected, no cipher below `[Minimum TLS policy]`; no unresolved critical or high findings; accepted medium findings owned, remediation date set; secrets in managed secret store, one rotation proven; in-scope data stores encrypted at rest and in transit; edge rule coverage equal to source, rule for rule; 100% role, principal and trust policies mapped to a target equivalent, no principal without one; effective permissions equal to source's, none added or lost; no account-boundary grant unaccounted for, each reproduced on target or a named-owner gap; each of the six attributes valued or asserted against the target's own configuration, none settled by an assumed default | Security Engineer |
| 3.6 TA-6 Resilience and failover | Zone and component failures survived; recovery measured, not assumed | Fault injection in target non-production, then production in hypercare: multi-zone, instance, database failover, timed; connection chain's idle behaviour — backend keepalive, load-balancer idle timeout, any network load balancer's own timeout, connection-tracking idle expiry | Failover and recovery times; error rate; long-lived connections, in-flight transactions across a failover; primary zone removal; target's point-in-time recovery window against source's, restore inside it; reported keepalive and timeout values; idle holds in three classes — short, medium, long against load-balancer timeout and backend keepalive — boundary each ends at | 100% failovers leave the site serving within `[RTO target]`, no data loss beyond `[RPO target]`, both met in two consecutive runs; zero manual intervention; error rate never above `[Failover error tolerance]`; target's recovery window at least source's, restore inside it landing at expected point; backend keepalive > load-balancer timeout > target timeout on 100% paths tested; each idle hold closed only at or after earliest boundary it crosses — zero connections dropped or reset before it, every re-established connection carrying its request, no user-visible error, no data loss; a path where target cannot reproduce source's ordering is a finding with a decision, never accepted silently | SRE / Observability Engineer, with the Data Engineer |
| 3.7 TA-7 Observability | Dashboard, alert and log inventory exists and fires before cutover | Inventory comparison against source dashboards, alerts and logs; fault injection proving each alert fires and routes; dashboard walkthrough by the on-call role | Alert equivalents with threshold and window; severity-1 alerts fire and notify; logs for application, access, edge, database and jobs; retention set; request correlation or tracing, sample transaction | 100% alerts proven to fire and route within `[Alert delivery target]`; retention explicitly set on every log destination; 100% go/no-go panels populated in rehearsal | SRE / Observability Engineer |
| 3.8 TA-8 DNS, TLS and CDN | Resolution, certificates, caching, SEO survive each change | DNS checks per change; resolution probes from public resolvers; per-host certificate validation; cache hit, miss and stale-content tests; redirect, canonical, sitemap and `robots.txt` checks against the P1 rule set; CDN configuration, cache policies and edge code rebuild work, re-tested (1.4) | TTL lowered and expired; certificate chain, expiry, coverage; sitemap referenced from `robots.txt`; self-referential canonical tags; asset caching and content versioning; no mixed content or certificate-name mismatch; cache-key inputs against source's — query string, headers, cookies, path, compression — default TTLs, cache-policy defaults, invalidation quota; CDN certificates in an accepted region; each DNS validation record left in place at issuance; storage addressing style — virtual-hosted or path-style — dotted bucket name matched by its certificate; signed-URL signing scope, credential-limited paths; object-store consistency against documented exceptions, never retried until visible | Cert checks pass with `[Min certificate validity]` and full host coverage; redirects and canonicals match source exactly; `robots.txt` and sitemap reachable, no disallow; zero stale-content defects; each change inside `[TTL propagation window]`; 100% cache-key inputs reproduced, every default, quota or cache-policy difference a finding with a decision; no port accepted, each rebuilt configuration re-tested before first step; zero signed-URL requests rejected for signing scope; zero retry-until-visible reliance; every DNS validation record still in place at final step | Cloud Platform Engineer, with the Application Owner |
| 3.9 TA-9 Rollback | Way back works, fits the window, covers post-cutover writes | Two P4 rehearsals in production-sized non-production: reversal before the dry run, then a simulated shift; writes reconciled into source by 3.11's procedure — named consistency mechanism, reconciliation report evidencing the reconciled record count, `[Conflict resolution rule]` under its named owner — the Data Engineer executing inside the Cutover Manager's rehearsal | Traffic returns; post-cutover writes reconciled; trigger list exercised | Both rehearsals consecutively in P4 inside `[Max rollback time]`; zero data loss; reconciled count equal to post-cutover writes; journeys pass on source; decision inside `[Decision timebox]` | Cutover Manager, with the SRE / Observability Engineer and Data Engineer |

The five integration points of TA-3, what changes at cutover and the area covering each:

| Third party | What changes at cutover | Covered by |
|---|---|---|
| Identity provider, where sign-in is federated | Redirect or callback URLs registered against the source hostname; the assertion path after the step | TA-3, post-move sign-in in TA-1 |
| Payment provider | Source-address allow-lists; webhook callback URLs; the confirmation path end to end | TA-3, payment journey in TA-1 |
| Email or notification provider | Sending allow-lists; bounce and webhook callback URLs | TA-3, outbound delivery in TA-1 |
| Analytics and tag managers | Consent, cookie-domain and cross-domain configuration; the collection endpoint's view of the new hostnames | TA-3, page-level tag firing in TA-1 |
| Any other external API | Source-address allow-lists; callback and redirect registration | TA-3 |

### 3.10 States every test area must handle

**Empty or absent.** **No source baseline:** captured before testing starts, or the comparison cannot proceed; where unavailable, gap recorded with a named owner and an alternative reference. **No source equivalent:** a named-owner gap tested on its own behaviour; unowned fails the phase.
**Failing.** Each state carries a decision and an action; where one is also a rollback trigger (5.2), precedence is fixed — pause, halt or postpone until a trigger fires; once one fires, the step or cutover is reversed.

| Failure state | Decision and action |
|---|---|
| Data mismatch (count, checksum, hash, schema, orphan row) before traffic | Halt shifting; re-sync the scope; re-validate; resume on a clean comparison |
| Data mismatch after a step began | Reverse the step — a rollback trigger (5.2); the source resumes as system of record; re-sync and re-validate before retrying |
| Error rate or latency beyond threshold, no trigger fired | Pause inside the window, diagnose, resume once cleared; never advance on an unresolved breach |
| A breach that met a trigger — `[Rollback error rate]` across two windows, or p95 above `[Rollback latency multiple]` | Reverse the step or cutover; never pause on a met trigger |
| Lag beyond `[Max replication lag at freeze]` at the freeze | Postpone, not reverse: no traffic has moved and there is no step to reverse; never accept a lagging target as system of record |
| Lag beyond `[Max replication lag at freeze]` after a step began | Reverse the step, then re-sync and re-validate |
| A certificate or DNS failure — invalid, expired or missing for any host, or resolution failing beyond `[DNS failure threshold]` of probes | Roll the weights back; fix issuance or resolution; re-enter the step. At or below `[DNS failure threshold]` of probes, observe and diagnose inside the window instead of reversing; the step never advances while it persists and the window may not close inside its thresholds |
| Third party rejecting target traffic | Fix the allow-list or callback registration before the next step; if not inside `[Third-party fix window]`, reverse |

**Ordinary.** The step passes, the smoke test and every threshold inside their limits, and the owner records the sign-off before the next step; nothing advances on verbal agreement.

### 3.11 Edge cases and their treatments

| Edge case | Stated treatment |
|---|---|
| Sessions and cookies split across the two environments | Session state outside the serving environment, or load-balancer stickiness, so either route keeps the session; a signed-in user stays signed in across a weight change, `SameSite` and cookie domain behaving; a forced sign-out is a defect |
| Cached DNS resolvers ignoring the lowered TTL | TTL lowered before the window and the previous TTL waited out; resolution probed from several resolvers; window sized to observed convergence; the way back reachable for the propagation window after a reversal |
| Data written to both environments during the parallel run | One write path authoritative, or dual writes reconciled by a named rule; the consistency mechanism proven before P3 exits, its reconciliation report signed off by the Data Engineer as 2.3's exit evidence; conflicts by `[Conflict resolution rule]`, named owner; reconciled count equals the dual-written count |
| Time-zone, character-encoding and collation | Time zone, character set and collation compared per database; non-ASCII and multi-byte content, sort order, date rendering across a daylight-saving boundary and a time-zone offset spot-checked; the settings, casing and recovery-window comparisons of TA-2 and TA-6 run explicitly, any restart-required change applied first, a shorter recovery window a named-owner gap never accepted silently; a difference beyond the accepted list is a defect |
| Large uploads, signed URLs and CORS rules depending on the storage domain | Signed-URL generation and consumption tested from the target within the provider's lifetime and after a key rotation; both environments cap a signed URL at seven days, so signing scope and credential-limited paths are what differ; the largest supported upload attempted; CORS preflight and the allow-list verified against the target hostname; the addressing-style, dotted-bucket, consistency-exception and retry-until-visible checks of TA-8; a failure halts the step and fails TA-3 and TA-8 |
| Scheduled jobs running twice, or not at all, during the switch | One environment owns each schedule, the other's disabled; run logs prove every job ran once per schedule, no missed run; jobs with side effects checked for idempotency; a duplicate or missed run is a defect at the cutover gate |

## 4. Cutover and hypercare testing

### 4.1 The smoke set run at every traffic step

Eight checks: pages succeed, no certificate warning, no maintenance page, each page's analytics tags firing once against the target hostnames; static assets; title, canonical, `robots` metadata; `robots.txt`, sitemap; authentication — sign-in, session establishment and sign-out as on the source, a signed-in session surviving the step; a revenue-critical action; one read-only admin action, a non-privileged role denied it; TA-3 handshakes.
A failed check stops the step: no window opens, nothing advances; re-run only once TA-1 or TA-3 confirms the fix. A certificate or DNS failure, or a third party rejecting target traffic, takes 3.10's matching row; no other smoke failure has one and is a functional defect at the cutover gate, as 3.11 treats a duplicate or missed job. Inside an open window a breach follows 3.10's pause rows; a condition that also met a trigger (5.2) is reversed, not paused.

### 4.2 Traffic-shift steps and monitoring windows

Steps **1%, 10%, 50%, 100%**; the target's DNS weight staged at zero, raised only at step 1, so nothing reaches the target before its smoke set passes. Weight, TTL and mixed-traffic mechanics: at 1.2.
Each step runs the smoke set, then holds `[Step monitoring window]` — at least one representative peak-traffic period, never under `[Max rollback time]`.
**Certificate preparation.** Every certificate a step presents covers every hostname it serves with `[Min certificate validity]` remaining; CDN certificate region and each DNS validation record left in place at issuance (TA-8).
A breach follows 3.10; a met trigger reverses the step; shift-window writes reconciled to an equal record count (TA-2, TA-9), every write made during the shift appearing in the reconciliation, any conflict by `[Conflict resolution rule]`; the Data Engineer records the written and reconciled counts the Cutover Manager signs the P5 exit against.

### 4.3 Hypercare

Hypercare runs `[Hypercare duration]` and exits by signal: issue volumes at or below the pre-cutover baseline; no highest-severity incident beyond `[Max P1 open age]`, none major beyond `[Max P2 open age]`; latency and error rate inside threshold (TA-4); the TA-6 production failover complete; source receiving no production writes, reconciliation done.
The source stays live and in sync, 5.2's triggers in force and TA-9's reversal ready, until then.

### 4.4 Downtime and the permitted change window

Production downtime during cutover: assumed minimal or none. The change window — duration, time of day, any freeze — is `[Permitted cutover window]`: confirmed in P4 and a go/no-go entry at 8.1.

## 5. Go/no-go checklist and rollback triggers

### 5.1 Go/no-go checklist for cutover

Every entry measurable and green before the first step; the decision is a signed checklist line.

1. P4 exit signed off; dry-run complete.
2. Source baseline captured; no journey without one.
3. Data clean: zero count, checksum, hash mismatches; zero orphan rows; lag inside `[Max replication lag at freeze]`, stable for `[Stability window]`.
4. Target smoke set (4.1) passing at zero DNS weight.
5. Observability: every alert proven to fire; retention set on every log destination.
6. Security: no unresolved critical or high findings; one rotation proven; encryption verified.
7. Resilience: RTO and RPO at or below `[RTO target]`/`[RPO target]`.
8. Integration: allow-list and callback re-registrations confirmed in writing.
9. Both rollback rehearsals inside `[Max rollback time]`, each with post-cutover writes.
10. TTL lowered; target weight staged at zero.
11. `[Permitted cutover window]` confirmed; freeze in effect; rota.

### 5.2 Rollback triggers

Any one reverses the step or the cutover; a fired trigger beats the pause path at 3.10.

1. A critical journey's error rate above `[Rollback error rate]` across two windows.
2. p95 above `[Rollback latency multiple]` of baseline for `[Rollback latency duration]`.
3. Any data integrity mismatch after the shift: count, checksum, hash, orphan row.
4. Lag beyond `[Max replication lag at freeze]` at the freeze, once a traffic step has begun.
5. A certificate invalid, expired or missing for any host, or DNS resolution failing beyond `[DNS failure threshold]`.
6. A security incident from the migration: exposure, a credential leak, or exploitation of a rule that should have blocked it.
7. A third party rejecting target traffic, not fixable inside `[Third-party fix window]`.
8. A fault undiagnosed and without a resolution path inside `[Decision timebox]`; default rollback.

### 5.3 The rehearsed rollback

Rehearsed twice consecutively in P4, before the first production traffic shift: the first reverses the target before the dry run, the second a simulated shift inside it, each timed against `[Max rollback time]`.
Each reconciles its writes to a count equal to the post-cutover write count (TA-9); exceeding `[Max rollback time]` fails P4; the reversal is a DNS weight change returning users to a source still in sync.

## 6. Roles, environments, risks and mitigations

### 6.1 Roles and responsibilities

Role names only: no team or person name appears anywhere in this plan.

| Role | Responsibility | Environments |
|---|---|---|
| Test Lead | Owns the plan and gates; signs phase exits | All three, plus control |
| QA / Test Engineer | Functional, regression and integration checks | Source (RO), non-prod, prod |
| Application Owner | Journeys, admin functions; owns TA-1, hypercare | Source, non-prod, prod |
| Cloud Platform Engineer | Owns TA-8, the platform build, decommission | Source (RO), non-prod, prod |
| Data Engineer | Owns TA-2: replication, validation, reconciliation | Source (RO), non-prod, prod |
| Performance Engineer | Owns TA-4: baseline capture, comparison | Source (approved load window), non-prod, prod |
| Security Engineer | Owns TA-5: policy, network, TLS, edge | Source (RO config), non-prod, prod |
| SRE / Observability Engineer | Owns TA-6 and TA-7: failover, alerts | Source (RO monitoring), non-prod, prod |
| Third-Party Integration Owner | Owns TA-3: allow-list and callback re-registration | Non-prod, prod, consoles |
| Cutover Manager | Owns TA-9, the traffic steps, go/no-go | Non-prod, prod, source, DNS control |
| Programme Stakeholder | Signs the gates against recorded exit criteria | None: review only |

### 6.2 Environments

| Environment | Definition | Used for |
|---|---|---|
| Source production | GCP environment serving live traffic | Baselines, comparison, rollback target |
| Target non-production, sized like production | Production-equivalent sizes and storage; no live traffic | Validation, rehearsal, dry run |
| Target production | AWS after cutover, serving live traffic | Hypercare, post-cutover types |

Two external control surfaces, recorded as control surfaces, never environments: the **DNS control plane** and **each provider's console**; producing this plan creates none of these environments.

### 6.3 External dependencies the plan relies on

| Dependency | Why the plan needs it | Owner role |
|---|---|---|
| Target cloud account with create rights | The target environments | Cloud Platform Engineer |
| Network between the clouds | The parallel run, reconciliation | Cloud Platform Engineer, Data Engineer |
| DNS control of the zones, with weights and TTLs | Traffic steps, cutover, rollback | Cutover Manager |
| Certificate issuance authority, including the DNS validation record and leaving it in place | Certificate validity and renewal on the target, which depends on that record (TA-8, 4.2) | Cloud Platform Engineer |
| Access to third parties whose allow-lists change | Integration re-registration | Third-Party Integration Owner |
| Read access to source monitoring and config | Baseline capture; source redirect and cache rules | Test Lead, Security Engineer, SRE / Observability Engineer, Cloud Platform Engineer |
| An approved load-testing window on source | The TA-4 baseline | Performance Engineer |
| Approval for fault injection in production | The TA-6 failover | SRE / Observability Engineer |
| Backup and restore tooling, restore authority | The measured RTO and RPO in TA-6 | Data Engineer |

This plan creates no infrastructure and opens no cloud account, so every dependency is a precondition satisfied later; and no dependency may be satisfied by putting live customer personal data in non-production (1.3).

### 6.4 Risk register and mitigations

| Risk | Mitigation | Test area |
|---|---|---|
| Silent data loss or corruption | Counts, checksums, schema parity, orphans | TA-2 |
| Functional regression on a critical journey | Baseline on source, replay on target | TA-1 |
| Scheduled jobs running twice or not at all | Single-owner schedule, run-log comparison | TA-1 |
| A third party rejecting traffic or stale callbacks | Re-register allow-lists and callbacks before the step | TA-3 |
| Performance regression or capacity shortfall | Baseline-then-compare on like scenarios | TA-4 |
| Autoscaling too slow or unstable for a spike | Observe scale-out timing and oscillation | TA-4 |
| Over-permissive identity, weak TLS, missing edge rule, plaintext secrets | Least privilege, handshakes, secret checks | TA-5 |
| Edge traffic reaching the target too early | Stage the DNS weight at zero; require a smoke pass | TA-8 |
| A certificate or DNS failure, or stale caching | Lower the TTL ahead, validate per host | TA-8 |
| SEO and URL regressions after the host change | Redirect, canonical, sitemap, `robots.txt` | TA-8 |
| A zone, instance or database failure taking the site down | Failover tests with measured RTO/RPO | TA-6 |
| Backups unrestorable inside the objectives | Restore into a clean environment and time it | TA-6 |
| Operating blind: dashboards, alerts or logs missing | Inventory parity plus a fault-injection alert proof | TA-7 |
| Rollback failing, exceeding the window, losing writes | Rehearse in production-like conditions and time it | TA-9 |
| Diverging data while both environments take writes | Prove the consistency mechanism; reconcile | TA-2, TA-9 |

## 7. Traceability

### 7.1 Test areas, the risks they address and the environments they run in

| Test area | Risks addressed | Environments |
|---|---|---|
| TA-1 Functional and regression | Functional regression; missed jobs | Source (RO), non-prod, prod |
| TA-2 Data integrity | Data loss or corruption; diverging data in the run | Source (RO), non-prod, prod |
| TA-3 Integration | A third party rejecting traffic; stale callbacks | Non-prod, prod |
| TA-4 Performance and load | Performance regression; capacity shortfall; unstable scaling | Source (approved load window), non-prod, prod |
| TA-5 Security | Over-permissive identity; exposed path; weak TLS | Source (RO config), non-prod, prod |
| TA-6 Resilience and failover | Zone, instance or database failure; unrestorable backups | Non-prod, prod |
| TA-7 Observability | Operating blind without dashboards or alerts | Source (RO monitoring), non-prod, prod |
| TA-8 DNS, TLS and CDN | Early traffic; cert or DNS failure; SEO loss | Source (RO), non-prod, prod |
| TA-9 Rollback | Rollback failure; over-long rollback; lost post-cutover writes; diverging data | Non-prod, prod, source |

Every area above reaches at least one risk, and every risk at 6.4 reaches at least one area: neither table contains an orphan.

### 7.2 Service-mapping coverage

Every row of the mapping table at 1.4 is covered by at least one test area.

| Mapping row | Covered by |
|---|---|
| Compute | TA-1, TA-4, TA-6 |
| Relational database | TA-2, TA-4, TA-6 |
| Object storage | TA-2, TA-8 |
| Load balancing and CDN | TA-4, TA-6, TA-8 |
| DNS and certificates | TA-8 |
| Identity and secrets | TA-5 |
| Observability | TA-7 |
| Edge security | TA-5, TA-8 |

A row that does not apply is recorded as a gap with a named owner rather than dropped; 3.10 states the consequence of leaving a gap unowned.

## 8. Assumptions to confirm

### 8.1 Assumptions

This final section lists the assumptions the plan rests on, not facts; each carries why it matters, and the document is revised as they are confirmed.

| Assumption | Why it matters |
|---|---|
| Dynamic site on the mapping table's services | Mapping drives the areas |
| Every mapping row assumed until confirmed | An inapplicable row changes an area's proof |
| Cutover is a parallel run with weighted DNS shift | Phase sequence and rollback rest on it |
| Steps 1%, 10%, 50%, 100%, each monitored | Smoke tests attach to steps |
| Like-for-like migration, no application changes | A change would invalidate the comparison |
| Downtime minimal or none; `[Permitted cutover window]` | Rollback triggers and timing depend on it |
| Hypercare duration `[Hypercare duration]` | Exit criteria are measured against it |
| Maximum rollback time `[Max rollback time]` | Bounds rehearsals and the timebox |
| Every other placeholder agreed before the phase that first uses it | A threshold a gate depends on |
| TTL lowered to `[TTL value]` 24-48 hours ahead | Shift works after the TTL expires |
| Source behaviour is the comparison reference | Without a baseline TA-4 cannot report |
| Anonymised production copy for test environments | Validation and comparison need it |
| Environments: source, target non-prod, target prod | Phases and owners map to these |
| One Markdown document, high level, no test cases | Boundary with the runbook, assets |
| Migration runbook exists and is owned elsewhere | Plan references it, never replaces it |
| Non-website workloads migrated separately | Deliberate absence, not omission |

### 8.2 The bracketed placeholders to be agreed

Each of the twenty-nine placeholders below is a value the programme has not yet supplied; none is replaced by an invented figure, and each is agreed before the phase that first depends on it.

- **Thresholds:** `[Agreed latency tolerance]`, `[Agreed error-rate tolerance]`, `[Max sustained utilisation]`, `[Max scaling events]`, `[Max scale-out time]`,
`[Max replication lag at freeze]`, `[Stability window]`, `[Failover error tolerance]`, `[RTO target]`, `[RPO target]`.
- **Cutover, DNS and TLS mechanics:** `[TTL value]`, `[TTL propagation window]`, `[Step monitoring window]`, `[Permitted cutover window]`, `[DNS failure threshold]`, `[Min certificate validity]`, `[Minimum TLS policy]`.
- **Operational windows and rollback thresholds** (including the incident-severity ages `[Max P1 open age]` and `[Max P2 open age]`): `[Alert delivery target]`, `[Max P1 open age]`, `[Max P2 open age]`, `[Rollback error rate]`, `[Rollback latency multiple]`, `[Rollback latency duration]`,
`[Max rollback time]`, `[Decision timebox]`, `[Third-party fix window]`, `[Conflict resolution rule]`, `[Decommission observation window]`, `[Hypercare duration]`.

Until a placeholder is agreed, the test area, gate or trigger that depends on it cannot be reported as passed.
