# GCP to AWS Website Migration Test Plan

## About this plan

**What it is.** A high-level test plan that a migration team follows to verify that a website moving from
Google Cloud Platform (GCP) to Amazon Web Services (AWS) loses no data, no functionality and no security
posture, keeps its performance inside the thresholds this plan sets, and has a rehearsed way back if the cutover
fails. It states what is tested, in which phase, by whom, against which pass threshold, and when the team must
roll back.

**What it is not.** The plan is a document, not a system, and producing it executes no test. Writing it creates
no infrastructure, provisions no environment, accesses no cloud account and contacts no third party. Every
environment, account and dependency it names is a precondition the migration team satisfies later, and nothing
in the plan claims otherwise. Building the target, porting rules and re-registering allow-lists belong to the
programme this plan serves, which this plan specifies and does not perform.

**Altitude.** The plan is approach level and targets a compact read of roughly five to ten pages: it names
test areas, methods, owners and pass thresholds, and it carries its detail in tables rather than prose. It
contains no step-by-step procedures, no test cases, no scripts, no automation code and no fenced code or
diagrams; a threshold table is in scope, an automation script is not. Its length is set by what the plan has to
cover — seven phases with entry and exit criteria, nine test areas, and the gates, thresholds and risk and
traceability tables — and per-test-case detail belongs to the test assets the programme maintains.

**Relationship to the migration runbook.** This plan references the migration runbook without replacing it. The
runbook holds the executed steps; this plan holds the phases, the gates, the thresholds and the rollback intent.
Where the two touch the same activity, the plan states what must be true and the runbook states how it is done.
The runbook is an external artefact owned by the migration programme, is not duplicated here, and is available
for reference during the pre-cutover rehearsal.

**Naming and placeholders.** Role names are used throughout — no team name, person name or fixed date appears,
and no tooling brand is mandated. Where a product is mentioned it is an example of its category, not a
requirement. Every value the programme has not yet agreed appears as a bracketed placeholder such as
`[Hypercare duration]`, and every placeholder is listed in section 8.

**Identifiers.** Migration phases are `P1` to `P7`; test areas are `TA-1` to `TA-9`. A phase or an area keeps the
same identifier in every section, block and table that refers to it.
Incident severities are written out in full rather than abbreviated — as highest-severity and major-severity in
2.6 and 4.3 — so `[Max P1 open age]` and `[Max P2 open age]` are incident-severity placeholders, not phase
identifiers.

**Scope in one line.** A like-for-like move of one website, tested in seven gated phases and nine test areas,
with a parallel run and a progressive weighted traffic shift, and a rehearsed way back to the source.

## 1. Strategy and approach

### 1.1 Test levels

Four levels are used, and each of the nine test types of this plan is applied at the level at which its result
can be judged.

| Level | Test types judged at this level | What the level proves |
|-------|--------------------------------|-----------------------|
| Smoke | A short cross-cutting set drawn from functional and regression, integration, and DNS/TLS/CDN checks | That the site a real user reaches is serving correctly, in the eight checks 4.1 defines: public pages that return a success status with no certificate warning and no maintenance page; static assets; the title, canonical and `robots` metadata; `robots.txt` and the sitemap; authentication; one revenue-critical action; one read-only administrative action; and the integration handshakes of TA-3 |
| Functional and regression (system level) | Functional and regression | That every critical user journey, administrative function, scheduled job and outbound message behaves on the target as it did on the source |
| Provider-contract integration | Integration | That every third-party system accepts the target's traffic and that no allow-list, callback, webhook or redirect still points at the source |
| System-level non-functional | Performance and load, security, resilience and failover, observability, DNS/TLS/CDN | That latency, capacity, security posture, failure behaviour, operability and naming behave on the target as required |

Two of the nine test types are deliberately not levels of their own:

- **Data integrity is a cross-environment comparison.** Source and target are compared with each other — counts, checksums, schema definitions, referential integrity, object counts and per-object hashes — rather than each environment being exercised alone.
- **Rollback is a rehearsal of the whole cutover sequence.** It exercises traffic reversal, integration, functional behaviour and data reconciliation together, which is why it is planned and timed as a rehearsal rather than as a level.

### 1.2 The cutover model this plan assumes

The cutover this plan tests is a **parallel run on the target followed by a progressive weighted DNS traffic
shift from the source to the target**. Both environments serve during the shift, the source stays live and in
sync and remains the system of record until the hypercare exit criteria in section 4 are met, and the source is
the primary way back.

| Stage | Target DNS weight | What happens | Branch if a threshold is breached |
|-------|-------------------|--------------|-----------------------------------|
| Parallel run | 0% | The source serves all live traffic and stays the system of record; the target is exercised directly and by synthetic checks; replication keeps the two environments consistent | No production traffic to reverse: fix the target and re-enter the stage; the first production step does not start |
| Step 1 | 1% | The smoke set runs, then the monitoring window of `[Step monitoring window]` | Reverse the DNS weights so the source serves all traffic again |
| Step 2 | 10% | The smoke set runs, then the monitoring window of `[Step monitoring window]` | Reverse the DNS weights so the source serves all traffic again |
| Step 3 | 50% | The smoke set runs, then the monitoring window of `[Step monitoring window]` | Reverse the DNS weights so the source serves all traffic again |
| Step 4 | 100% | The smoke set runs, then the monitoring window of `[Step monitoring window]` | Reverse the DNS weights so the source serves all traffic again |
| Hypercare | 100% to the target | Hypercare runs for `[Hypercare duration]`; the source stays live and in sync and the way back stays available | Reverse the DNS weights while the way back is still available; the source is still in sync |

A traffic step advances only when its monitoring window has closed inside every threshold and the step's
sign-off has been recorded by the named owner; the parallel run hands over to the first traffic step only when
the go/no-go checklist in section 5 is green, and hypercare ends on the exit signals in 4.3. Section 4 states
the smoke set, the steps and the windows. Nothing in this plan advances on a verbal agreement.

Weighted records are relative, and every record in a weighted group shares one TTL, so a step is advanced by
changing weights and never by editing addresses. The TTL is lowered to `[TTL value]` at least 24-48 hours
before the window opens, and the shift waits out the previous TTL before the first step. Each weight change
then takes effect within `[TTL propagation window]` (TA-8), so a period of mixed old and new traffic follows
every step; that is expected and tested rather than treated as an anomaly. These mechanics are stated once,
here, and section 4 does not restate them.

Downtime during cutover is assumed to be minimal or none, and the permitted change window — its duration, its
time of day and any freeze it sits inside — is `[Permitted cutover window]`, to be confirmed before the go/no-go
decision (section 5).

### 1.3 Test data

Data migration validation (TA-2), functional regression (TA-1) and the performance comparison (TA-4) all require
an **anonymised production copy** with realistic volume and distribution, so that row counts, indexes, cache
behaviour and query plans are representative.

This is a requirement of the plan, not a preference:

- Live customer personal data must not be used in any non-production environment, including the target rehearsal environment.
- Redaction or tokenisation is completed before the copy enters the rehearsal environment.
- A test area that cannot obtain an anonymised production copy cannot claim its result; the shortfall is recorded with a named owner as an exit-criteria failure for the phase it affects, never quietly passed.

### 1.4 Source-to-target service mapping

| Concern | Source (GCP) | Target (AWS) |
|---------|--------------|--------------|
| Compute | Compute Engine, GKE, Cloud Run | EC2, EKS, ECS Fargate |
| Relational database | Cloud SQL | RDS, Aurora |
| Object storage | Cloud Storage | S3 |
| Load balancing and CDN | Cloud Load Balancing, Cloud CDN | ALB, CloudFront |
| DNS and certificates | Cloud DNS, Google-managed certificates | Route 53, ACM |
| Identity and secrets | IAM, Secret Manager | IAM, Secrets Manager |
| Observability | Cloud Monitoring, Cloud Logging | CloudWatch, X-Ray |
| Edge security | Cloud Armor | AWS WAF, Shield |

The website's stack has not been described to this plan, so this table is the assumed mapping that the plan
tests rather than re-decides. Because the stack is unknown, **no row can be excluded outright** and **every row
is an assumption to confirm** (section 8). The plan makes no claim about the website beyond these rows.

This table appears once, in this section. Every row is covered by at least one test area, and that coverage is
checked in section 7. Two rows are flagged here because their applicability changes what a test area has to
prove:

- **Edge security.** If the source has no edge firewall or DDoS protection today, this row is not a port but new configuration on the target. The difference is recorded as a gap with a named owner, and TA-5 and TA-8 then test the new configuration against its own stated behaviour instead of against source parity.
- **Identity and secrets.** If the website has no end-user identity provider, this row reduces to workload and platform identity plus secret storage, and the post-move authentication assertions in TA-3 then cover internal service-to-service calls only.

### 1.5 Like-for-like

The migration is like-for-like: no re-architecting, no feature change, no service substitution beyond the
mapped pair and no cost-driven redesign. Where the target cannot reproduce a source behaviour exactly — an idle
timeout, a cache default, an edge-function capability, a collation — the difference is recorded as a finding
with a decision. It is not licence to redesign the website. Where the target's configuration model differs
rather than only its defaults, the configuration is rebuilt and re-tested on the target instead of ported: the
CDN configuration, its cache policies and any edge code are rebuilt and re-tested against the source's observed
behaviour before cutover (TA-8).

### 1.6 What this plan requires of the migration programme

1. **All nine test types.** Functional and regression, data integrity, integration, performance and load, security, resilience and failover, observability, DNS/TLS/CDN, and rollback are all in scope, and each is described as a test area in section 3.
2. **A dry-run cutover in a production-like target environment before the real one.** The real cutover does not start until a dry run has completed there with the smoke set passing at every simulated traffic step (P4, section 2).
3. **The rollback rehearsed twice, consecutively, before the first production traffic shift.** Both rehearsals run in P4, each covers data written to the target after the shift, each is timed end to end, and each must complete inside the stated maximum rollback time `[Max rollback time]`. A rehearsal that exceeds it fails P4 and the cutover does not proceed.

### 1.7 Conventions that govern this plan

- **Phased and gated.** The plan spans seven phases in order (section 2). No phase starts until the previous phase's entry criteria are met and its exit criteria are signed off by the named owner role.
- **Baseline-then-compare.** The source is measured first. Functional, integration and performance results on the target are judged against a captured source baseline, never against a recollection of how the site used to behave.
- **Rollback-first.** The way back is designed, rehearsed and timed before the first production traffic shift, including the treatment of data written to the target after cutover, and the source keeps serving and receiving replicated change until the hypercare exit criteria are met.
- **One consistent test-area format.** Every test area in section 3 states objective, method, example checks, pass threshold and owner, in that order, so any two areas can be compared at a glance.
- **Every threshold objectively checkable.** Each test area carries at least one quantified pass threshold — a count, a percentage, a latency, an error rate or a lag in seconds — and every threshold, go/no-go entry and rollback trigger is a number, a comparison against the captured source baseline, or a binary gate such as a signed exit criterion or a freeze in effect. Where the programme has not agreed a value, the plan carries a bracketed placeholder rather than a figure invented on the user's behalf.
- **Evidence at every gate.** An exit criterion is satisfied by a recorded artefact — a report, a dashboard capture, a signed checklist line — and by nothing else.

### 1.8 Tool categories and their limits

The plan needs capabilities, not products, so tooling is named by category. A product mentioned anywhere in
this plan is an example of its category, never a requirement.

| Category | Test areas served | What it can verify | What it cannot verify |
|----------|-------------------|--------------------|-----------------------|
| Load generation | TA-4 | Latency percentiles, throughput, error rates and the shape of the response under normal, peak and spike profiles | Data correctness, authorisation logic and rendering quality |
| Browser and API regression automation | TA-1, TA-3 | Critical user journeys and API responses across browsers and viewports, and the presence of expected page state | Infrastructure timeouts, firewall or NAT behaviour, and behaviour under load |
| Data comparison | TA-2 | Row-by-row, checksum and aggregate comparison between source and target, inside documented engine limits such as a required primary key | Business-rule correctness and application behaviour that depends on the data |
| Vulnerability scanning | TA-5 | Known-vulnerability findings in packages and images, and exploitable web-application issues found by active probing | Identity-policy equivalence and business logic, and the parts of the estate the scanner never reaches |
| Synthetic monitoring | TA-7, TA-8 | Scheduled checks that a scripted journey keeps working from outside the environment, with an alert when it stops | Anything outside the scripted paths, and anything under load |
| DNS and TLS inspection | TA-8 | Resolution and TTL convergence, certificate chain and expiry, and the handshake versions actually accepted | Application behaviour behind the name, and third-party allow-list state |

For illustration only, and never as a mandate: load generation (for example k6, Apache JMeter or Locust),
browser and API regression automation (for example Playwright, Cypress or Selenium), vulnerability scanning (for
example Trivy or ZAP), data comparison (for example a managed migration service's row validation or an
open-source row-diff tool), synthetic monitoring (for example a managed canary service), and DNS and TLS
inspection (for example a public resolver probe service or a TLS scanner).

No category can prove identity-policy equivalence, allow-list re-registration, data reconciliation or the
correctness of a failover decision. Those remain evidence-backed manual checks with a named owner, and the plan
says so wherever it relies on one.

### 1.9 Scope boundaries

**In scope.** The seven phases and their gates; the nine test areas with their methods and thresholds; the
service-mapping table and its coverage; the cutover, its traffic steps and their monitoring windows; hypercare;
the go/no-go checklist and the rollback triggers; the rollback rehearsals; the roles, environments and risk
register; the traceability tables; and the assumption list.

**Out of scope**, named here so the boundary is explicit:

- Detailed test cases, scripts and automation code — this plan names the areas and the approach only.
- Re-architecting or feature changes to the website, and any cost optimisation or FinOps analysis: the migration is like-for-like.
- The migration runbook itself, which this plan references but does not replace.
- Migration of non-website workloads, such as internal tools and analytics pipelines.
- Building the target environment, porting edge rules and re-registering allow-lists: this plan states what must be true at each gate, and that work belongs to the migration programme.
- Any user interface, component library, design system or design asset: none was supplied and none is produced.


## 2. Phase-by-phase test activities

Seven phases run in order. No phase starts until the previous phase's entry criteria are met and its exit
criteria have been signed off by the named owner role, and every exit criterion is satisfied by a recorded
artefact (section 1.7). Each block below states that phase's entry criteria, its exit criteria and its owner
role.

**The test types run in each phase are stated once, in the test-types table at 2.8, which is the single
statement of that fact in this plan.** Each phase block points to that table rather than repeating its content,
and no other section or table in this document restates it.

### 2.1 P1 Pre-migration baseline

**Owner role:** Test Lead.

**Entry criteria.**

- Source production is identified and accessible to the test team.
- The roles named in this plan are assigned.
- An anonymised production copy can be produced (section 1.3).

**Exit criteria.**

- Source functional baseline recorded for every critical user journey.
- Source performance baseline captured under normal, peak and spike profiles.
- Source data profile recorded: per-table row counts, checksums, object counts and hashes.
- Current identity, network, TLS, edge firewall, secrets and encryption configuration inventoried.
- Existing dashboards, alerts and logs inventoried, each with its threshold.
- Replication from source to target established, with its initial sync verified.
- Every critical journey scripted or documented as a manual check.

**Test types run in this phase:** stated in 2.8.

### 2.2 P2 Target environment validation

**Owner role:** Cloud Platform Engineer.

**Entry criteria.**

- P1 exit criteria signed off.
- The target non-production environment is built and sized like production.
- The test team has access to it.
- Connectivity between the two clouds is in place.

**Exit criteria.**

- The target serves the smoke set successfully, with no certificate warning and no placeholder content.
- Target observability exists, and every alert has been proven to fire in a fault-injection test.
- Identity least privilege, network segmentation, TLS policy, edge firewall rules, secrets handling, and encryption at rest and in transit are verified on the target.
- Resilience rehearsed in the target — multi-zone failover, instance and database failover, backup and restore — with RTO and RPO measured.
- DNS and certificate readiness verified in non-production.
- Every gap where a target component has no source equivalent is recorded with a named owner.

**Test types run in this phase:** stated in 2.8.

### 2.3 P3 Data migration validation

**Owner role:** Data Engineer.

**Entry criteria.**

- P2 exit criteria signed off.
- Replication is running.
- An anonymised copy is available on both sides for comparison.

**Exit criteria.**

- Per-table row counts match.
- Per-table checksums or aggregates match.
- Schema definitions match — tables, indexes, constraints, sequences and triggers — apart from a documented list.
- Zero orphan rows across all child-to-parent checks.
- Object counts and per-object hashes match.
- Replication lag on both the source and target sides is below the agreed limit of `[Max replication lag at freeze]` and has been stable for the agreed stability window of `[Stability window]`.
- Every mismatch is investigated, resolved, or recorded as an accepted difference with an owner.
- The mechanism that keeps the two environments consistent during the parallel run is named and proven.

**Test types run in this phase:** stated in 2.8.

### 2.4 P4 Pre-cutover rehearsal

**Owner role:** Cutover Manager.

**Entry criteria.**

- P3 exit criteria signed off.
- A production-like target environment is available.
- The migration runbook is available for reference.
- The dry-run window is agreed.

**Exit criteria.**

- A dry-run cutover has completed in the production-like target with the smoke set passing at every simulated traffic step.
- The rollback has been rehearsed twice consecutively in this phase — the first reversal against the pre-cutover target, the second reversing a simulated traffic shift inside the dry-run cutover — each completed inside `[Max rollback time]` and each including data written to the target after the shift.
- Third-party allow-lists and callback, webhook and redirect URLs have been re-registered and verified.
- The go/no-go checklist (section 5) has been walked and every entry confirmed measurable.
- The permitted cutover window `[Permitted cutover window]` is confirmed.
- The hypercare duration `[Hypercare duration]` is agreed and the support rota is named.

**Test types run in this phase:** stated in 2.8.

### 2.5 P5 Cutover

**Owner role:** Cutover Manager.

**Entry criteria.**

- P4 exit criteria signed off.
- Every entry on the go/no-go checklist is green.
- The change freeze is in effect.
- The TTL has been lowered and the previous TTL has expired.
- The target's DNS weight is staged at zero.
- The support rota is staffed.

**Exit criteria.**

- Each traffic step's smoke test passed and its monitoring window closed inside every threshold, with the step's sign-off recorded before the next step began.
- 100% of production traffic is served by the target.
- Data written during the shift is accounted for and reconciled.
- The target is observable and alerting at the new traffic level.
- Every rollback trigger (section 5) was monitored against its threshold throughout the shift and is confirmed not breached.

**Test types run in this phase:** stated in 2.8.

### 2.6 P6 Post-cutover hypercare

**Owner role:** Application Owner.

**Entry criteria.**

- P5 exit criteria signed off.
- Traffic is fully shifted.

**Exit criteria.**

- Hypercare has run for the agreed duration with issue volumes back to the pre-cutover baseline.
- No highest-severity incident has been open longer than `[Max P1 open age]`, and no major-severity incident longer than `[Max P2 open age]`.
- Latency and error rate have stayed within threshold for the hypercare duration.
- The source no longer receives production writes and the final reconciliation is complete.
- Every open issue is documented with an owner and a next action.

**Test types run in this phase:** stated in 2.8. Hypercare exits by signal rather than by date; the signals and the monitoring behaviour are in section 4.

### 2.7 P7 Source decommission readiness

**Owner role:** Cloud Platform Engineer.

**Entry criteria.**

- The hypercare exit criteria are met and signed off by the Application Owner and the Cloud Platform Engineer.

**Exit criteria.**

- No production traffic reaches the source across an observation period of at least `[Decommission observation window]`.
- Reconciliation proves no source-only data and no unresolved write conflicts.
- The rollback decision is formally closed, so the way back is no longer in use.
- The source credentials, firewall rules, DNS records, monitoring and third-party allow-list entries are identified for removal, each with an owner and a removal checklist, with no date fixed by this plan.
- The source configuration is archived for audit.

**Test types run in this phase:** stated in 2.8.

### 2.8 Test types run per phase

This is the single statement in this plan of which test types run in which phase. It is not repeated elsewhere,
and each phase block above points here.

| Phase | Test types run |
|-------|----------------|
| P1 Pre-migration baseline | Functional and regression (baseline capture on the source); performance and load (source baseline); data integrity (source profile capture); security (identity, network, TLS, edge firewall, secrets and encryption inventory); observability (inventory of existing dashboards, alerts and logs); DNS/TLS/CDN (the source's redirect and canonical rules, cache behaviour and certificate coverage, captured as the reference rule set) |
| P2 Target environment validation | Functional and regression (smoke set); security; resilience and failover; observability (alert proofs); DNS/TLS/CDN (non-production readiness); integration |
| P3 Data migration validation | Data integrity; integration (the replication and comparison path); performance and load (behaviour at production data volume) |
| P4 Pre-cutover rehearsal | Functional and regression; integration; rollback; DNS/TLS/CDN; observability; performance and load; security (re-verification on the production-like target); resilience and failover (timed inside the dry run) |
| P5 Cutover | Functional and regression (smoke at each step); DNS/TLS/CDN; observability; data integrity (writes during the shift); integration; rollback (triggers monitored throughout) |
| P6 Post-cutover hypercare | Functional and regression; performance and load (production behaviour); data integrity (reconciliation); integration; observability; resilience and failover (production failover of the live service); DNS/TLS/CDN (cache, redirect and certificate behaviour at full traffic); rollback (readiness of the way back maintained until the hypercare exit) |
| P7 Source decommission readiness | Data integrity (final reconciliation); functional and regression (the source stays testable until the rollback window closes); security (identity and access removal review) |

Every test type this plan covers appears in at least one phase above.

## 3. Test areas

Nine test areas cover the nine test types. Each is stated in the same five fields in the same order —
**objective, method, example checks, pass threshold, owner** — so any two areas can be compared at a glance.
Which test types run in which phase is stated once, in the test-types table at 2.8, and is not restated here.
Where a block below names a phase, it places the check that block describes, not that phase's test-type set.

### 3.1 TA-1 Functional and regression

**Objective.** Prove every critical user journey, administrative function, scheduled job and outbound message
behaves on the target as it did on the source.

**Method.** Scripted browser journeys and API checks executed against the source to capture the baseline and
then against the target, replaying the same steps; administrative actions performed by an administrator role,
and a non-privileged role attempting the same action; authentication exercised with the same accounts and roles
on both sides, covering sign-in, session establishment, session lifetime, session renewal and sign-out;
page-level tag firing and the analytics requests each page issues observed on both hostnames; scheduled jobs
verified by their outputs and run logs; transactional email and webhooks verified by delivery to a test endpoint
that records receipt, order and signature.

**Example checks.** Browse; search; login and sign-out, with session establishment, session lifetime and session
renewal compared with the source; checkout or form submission; account management; admin console actions; a
non-privileged role denied an administrative action; page-level tag firing once per page view with the expected
identifiers on the target hostnames, with the analytics request the page issues resolving against them;
scheduled job output and job-run count; transactional email delivery; webhook receipt with signature validation;
error-page behaviour for a deliberately invalid request.

**Pass threshold.** 100% of scripted critical journeys pass on the target with responses equivalent to the
source baseline; zero differences from the source baseline in sign-in, session establishment, session lifetime
or sign-out behaviour; 100% of role-based access checks pass, including the negative case in which a
non-privileged role is denied an administrative action; zero failed administrative actions; every page view
fires its expected tags exactly once with the expected identifiers on the target hostnames, with zero duplicate
or missing firings; every scheduled job produces its expected output exactly once per schedule, with zero
duplicates and zero missed runs; 100% of test emails and webhooks delivered and acknowledged, with zero
duplicate sends.

**Owner.** Application Owner, executed by QA engineers.

### 3.2 TA-2 Data integrity

**Objective.** Prove no data was lost, corrupted, duplicated or left behind, and that both environments agree
while the parallel run is in progress.

**Method.** Layered comparison — counts, then checksums or aggregates, then schema definitions, then
referential integrity, then application-level reads — with object storage compared by inventory listing and
per-object hash, and replication lag monitored from both the source and the target side continuously up to and
through the write freeze.

**Example checks.** Per-table row counts; per-table checksums or aggregate comparisons; definition-level
comparison of tables, indexes, primary and unique constraints, sequences and triggers; child-to-parent
anti-joins for orphan rows; object counts and per-object hashes per prefix; replication lag at the freeze;
character-encoding and collation spot checks on non-ASCII and multi-byte content; a definition-level
comparison of the engines' settings and parameter groups, naming every setting that is immutable
after instance creation, every setting that cannot be edited in the target's default parameter
group, and every change that needs a restart to take effect, together with table-name and identifier
case sensitivity.

**Pass threshold.** Zero row-count mismatches; zero checksum mismatches; zero schema-definition differences
outside the documented accepted list; zero orphan rows; zero object-count or hash mismatches; replication lag
below `[Max replication lag at freeze]` on both sides and stable for at least `[Stability window]` before the
freeze; record counts for every reconciled write equal to the record counts written during the parallel run;
zero unrecorded database-settings differences, with 100% of the immutable-after-creation,
default-parameter-group and restart-required differences identified before P3 exits and either
matched on the target or recorded as an accepted difference with a named owner, and zero table-name or
identifier casing differences outside the documented accepted list.

**Owner.** Data Engineer.

### 3.3 TA-3 Integration

**Objective.** Prove every third-party system accepts traffic from the target, and that no allow-list entry,
callback, webhook or redirect still points at the source.

**Method.** Contract-level checks from the target's egress addresses against each provider — third-party APIs,
payment, email and identity providers, and analytics and tag managers —; verification of each re-registration
against the target hostnames; payload and response-code comparison between the source and the target calls,
with the source side replayed from the source baseline captured in P1 rather than by new traffic to source
production; negative tests that retired source addresses and old callback URLs are genuinely no longer accepted
or called; confirmation that the IP allow-lists and callback URLs which change with the new target egress
addresses have been re-registered. Two behaviours are asserted rather than assumed after the move.
**Authentication and authorisation:** sign-in, session establishment, session lifetime, sign-out and role-based
access behave on the target as they do on the source, including for administrative users, and the checks and
thresholds for them are the authentication checks of TA-1. **Data exchange:** each integration's requests are
accepted from the target's egress addresses, each callback, webhook or redirect the third party calls resolves
against the target hostnames, and the payload shape and response codes are unchanged. Whether a callback still
arrives at the source is read from each provider's own delivery logs, so the source side of every comparison
here is the P1 baseline and this area writes nothing to a source environment.

**Example checks.** Identity provider assertion and callback; payment authorisation and confirmation in the
provider's test mode; outbound email sending and bounce handling; webhook receipt and signature; analytics event
ingestion from the target hostnames; the analytics and tag-manager consent state, cookie domain and cross-domain
settings resolved against the target hostnames, with the collection endpoint seeing requests from them; a
retired source-address allow-list entry tested to confirm removal; an old callback URL tested to confirm it is
no longer registered; the storage hostname a third party calls back to (an upload callback or a signed-URL
redirect) resolves to the target and presents a certificate matching that name; each provider's delivery logs
read to confirm no callback still arrives at the source.

**Pass threshold.** 100% of integration endpoints reachable and functional from the target; zero provider
rejections attributable to a source address; 100% of callback, webhook and redirect registrations verified
against the target hostnames; zero residual source-address allow-list entries; zero callbacks recorded arriving
at the source after the final traffic step; zero consent, cookie-domain or cross-domain defects in the analytics
and tag-manager configuration, with 100% of tag and collection requests resolving against the target hostnames.

**Owner.** Third-Party Integration Owner, with a Cloud Platform Engineer for the network path.

The integration points this area covers, and what changes at cutover for each:

| Third party | What changes at cutover | Test area covering it |
|-------------|------------------------|-----------------------|
| Identity provider, where the website federates sign-in | Redirect or callback URLs registered against the source hostname; the identity assertion path after the traffic step | TA-3, with the post-move sign-in journey asserted in TA-1 |
| Payment provider | Source-address allow-lists; webhook callback URLs; the payment confirmation path end to end | TA-3, exercised through the checkout or payment journey in TA-1 |
| Email or notification provider | Source-address allow-lists for sending; bounce and webhook callback URLs | TA-3, with outbound email and webhook delivery asserted in TA-1 |
| Analytics and tag managers | Consent, cookie-domain and cross-domain configuration; the collection endpoint's view of the new hostnames | TA-3, with page-level tag firing asserted in TA-1 |
| Any other external API | Source-address allow-lists and any callback or redirect registration | TA-3 |

### 3.4 TA-4 Performance and load

**Objective.** Prove the target meets the source baseline under normal, peak and spike load and scales as
designed.

**Method.** Identical scenarios run against the source first and the target second — same data volume, same
journey mix, same think times, same durations — recording p50, p95 and p99 latency, throughput, the error mix,
resource saturation and every autoscaling event. The spike profile escalates abruptly to test the scaling
reaction, and the run continues until the system returns to baseline.

**Example checks.** p95 and p99 per critical journey; error mix by status class, separating client errors,
server errors, timeouts and throttles; throughput at peak; time from load onset to scale-out completion and the
number of instances or tasks added; behaviour after the spike passes, including scale-in; database connection,
CPU and storage saturation; queue or backlog depth where the website uses asynchronous work.

**Pass threshold.** p95 latency within `[Agreed latency tolerance]` of the source baseline for every critical
journey; error rate no higher than the source baseline plus `[Agreed error-rate tolerance]`; peak throughput at
least the source baseline; 100% of scale-out events complete inside `[Max scale-out time]` without a sustained
error-rate breach; sustained resource saturation below `[Max sustained utilisation]`; no more than
`[Max scaling events]` during the spike window, so the service returns to a stable footprint rather than
oscillating.

**Owner.** Performance Engineer, with the SRE / Observability Engineer.

### 3.5 TA-5 Security

**Objective.** Prove the target's security posture is at least equal to the source's, with no new exposure
introduced by the move.

**Method.** Policy and effective-permission review; configuration inspection; active TLS handshake testing;
dependency and image vulnerability scanning; runtime web scanning; secrets and encryption verification; and
negative tests for paths that must not work.
Identity policy is walked policy by policy, mapping each source policy statement to its target equivalent, and
the ported edge rules are staged in a non-blocking or count-only mode before any rule is enforced.

**Example checks.** Least-privilege review of every role and principal against the functions it serves, with no
wildcard permissions on production resources; network segmentation, with no internet path to a service except
through the edge; the TLS versions and ciphers actually accepted; HSTS and certificate subject-alternative-name
coverage per host; edge firewall rule parity against the source's rule set; secrets absent from code,
configuration and images and readable only by the intended identity, with a rotation exercised at least once;
encryption at rest and in transit for database, object storage and backups; vulnerability scan of images and
dependencies; administrative access path review, including multi-factor enforcement where the source enforced
it.
A policy-by-policy walkthrough of every identity policy maps each source policy statement to its target
equivalent and names the explicit deny precedence, organisation-level guardrail, permission boundary and trust
policy differences it carries, together with the additive bindings, deny policies, inheritance and impersonation
the target side brings, so every difference that changes effective access is asserted manually against recorded
evidence with the Security Engineer as the named owner; the account-boundary case exercises a principal whose
source access came from a folder- or organisation-level grant on the target to prove the same effective access
survives the boundary, with any lost grant named and replaced or else recorded as a gap with a named owner;
the ported edge rules run in a count-only or non-blocking mode first, and their match records are compared with
the source's observed behaviour before any rule is enforced; and the six named rule attributes are compared
one by one — managed rule sets, request-body inspection limits, the handling of over-long query strings,
rate-based rule thresholds, ban windows and attachment points.

**Pass threshold.** Zero wildcard or over-privileged grants on production resources; 100% of unintended
internet-reachable ports closed in the tested range; 100% of TLS 1.0 and 1.1 handshakes rejected and no cipher
below `[Minimum TLS policy]` accepted; zero unresolved critical or high findings at the gate, with any accepted
medium finding recorded with an owner and a remediation date; 100% of secrets held in the managed secret store
with one successful rotation proven; 100% of in-scope data stores proven encrypted at rest and in transit; edge
firewall rule coverage at least equal to the source, rule for rule.
100% of role, principal and trust policies walked and mapped to a target equivalent, with zero principals left
without a documented equivalent; every principal's effective permission set equal to its source effective set
for the same function, with zero principals holding access the source did not grant and zero source-effective
permissions lost; zero folder- or organisation-level grants unaccounted for at the account boundary, each
reproduced on the target or recorded as a gap with a named owner;
100% of ported edge rules run in a non-blocking or count-only mode first, with their match records compared
against the source's observed behaviour before enforcement and zero rules moved to blocking with an unexplained
difference; every one of the six named attributes listed with its source and target value, or with the behaviour
the target's own configuration is asserted against where the source has no counterpart, with zero differences
unaccounted for and none settled by a target default chosen by assumption.

**Owner.** Security Engineer.

### 3.6 TA-6 Resilience and failover

**Objective.** Prove the target survives component and zone failures, and that its recovery objectives are
measured rather than assumed.

**Method.** Controlled fault injection, first in the target non-production environment and then in production
during hypercare: multi-availability-zone failover, instance or host failure and replacement, and database
failover, with backup and restore into a clean environment. Each test is timed end to end, with the
user-visible error rate captured throughout, and the connection chain's idle behaviour is measured as
well: the backend keepalive, the load balancer's idle timeout, the timeout of any network load balancer
in the path, and the connection-tracking idle expiry.

**Example checks.** Multi-zone failover time and the error rate observed during it; database failover time and
connection recovery; replacement of a failed instance and return to service; restore of a backup into a clean
environment; measured RTO and measured RPO; the point-in-time recovery window achievable on the
target and the equivalent window on the source, and a restore to a chosen point inside the target's
window; behaviour of long-lived connections and in-flight transactions
across a failover; behaviour when the primary zone is deliberately removed from service; the values
actually reported for the backend keepalive, the load balancer's idle timeout and the target
application's own timeout, together with the connection-tracking idle expiry; connection paths that
include a network load balancer, whose own idle timeout is measured alongside the load balancer's; idle
connections held in three classes — short (shorter than the load balancer's idle timeout), medium
(between the load balancer's idle timeout and the backend keepalive) and long (longer than the backend
keepalive) — with the idle boundary at which each hold ends and the connection is reused or
re-established recorded.

**Pass threshold.** 100% of failover tests complete with the website serving again within `[RTO target]` and no
data loss beyond `[RPO target]`; measured RTO and RPO at or below those targets in two consecutive runs; the
point-in-time recovery window achievable on the target at least equal to the source's, with a
restore to a point inside it landing at the expected point and zero data loss beyond the source's
achievable recovery interval; zero
manual intervention required in the failover path; error rate during failover never sustained above
`[Failover error tolerance]` for longer than the failover itself; the reported values satisfy backend
keepalive > load-balancer timeout > target timeout on 100% of the connection paths tested; each idle hold
is closed only at or after the earliest idle boundary in the chain that it crosses — the target's own
timeout first, then the load balancer's idle timeout, then the backend keepalive — so zero connections
are dropped, reset or re-established before that boundary, and every connection re-established after an
expected close carries its request successfully with zero user-visible errors and zero data loss; any path
where the target cannot reproduce the source's ordering is recorded under section 1.5 as a finding with a
decision rather than accepted silently.

**Owner.** SRE / Observability Engineer, with the Data Engineer for database and restore tests.

### 3.7 TA-7 Observability

**Objective.** Prove the target can be operated: the dashboards, alerts and logs the migration depends on exist
and actually fire before cutover.

**Method.** Inventory comparison against the source's dashboards, alerts and log sources; fault injection to
prove that each alert fires and reaches the right destination; log completeness and retention checks; a
dashboard walkthrough by the role that will be on call.

**Example checks.** Every source alert has a target equivalent with a stated threshold and evaluation window;
each severity-1 alert fires in a fault-injection test and notifies the right destination within
`[Alert delivery target]`; logs present for application, access, edge, database and job activity at the new
traffic level; log retention set explicitly on every destination; the dashboard panels the go/no-go decision
reads are populated; request correlation or tracing available for a sample end-to-end transaction.

**Pass threshold.** 100% of the alerts on the agreed inventory created and proven to fire in a fault-injection
test; 100% routed to a monitored destination within `[Alert delivery target]`; retention explicitly set on every
log destination; 100% of the go/no-go dashboard panels populated during the rehearsal.

**Owner.** SRE / Observability Engineer.

### 3.8 TA-8 DNS, TLS and CDN

**Objective.** Prove name resolution, certificate validity, caching and SEO-critical behaviour survive the host
change, at every traffic step.

**Method.** DNS inspection before, during and after each weight change, including resolution probes from
multiple public resolvers; per-host certificate validation; cache behaviour tests for hit, miss and stale
content; redirect and canonical checks against the source rule set captured in P1; `robots.txt` and sitemap
validation; the CDN configuration, its cache policies and any edge code treated as rebuilding work rather than a
port and re-tested against the source's observed behaviour; cache-key composition, managed default TTL values,
cache-policy defaults and invalidation quotas compared with the source's observed values; the region a
certificate attached to a CDN distribution is issued in is checked against the regions that distribution
accepts; the DNS validation record left in place after issuance and confirmed still present, so renewal
continues; the object-storage addressing style in use, virtual-hosted or path-style, recorded and checked, with a
bucket name containing a dot matched by the certificate presented for that name; the signed-URL signing scope and
the credential-limited paths inside the target's model; and object-storage consistency judged against the
documented exceptions, because retry-until-visible assertions are obsolete.

**Example checks.** TTL lowered ahead of the window with the previous TTL observed to have expired; each weight
change fully reflected within the propagation window; certificate chain, expiry and subject-alternative-name
coverage per host; HTTP-to-HTTPS and canonical-host redirects; self-referential canonical tags; sitemap
reachable, well formed and referenced from `robots.txt`; `robots.txt` unchanged in effect; static asset caching
and content versioning; no mixed content and no certificate-name mismatch on any host; cache-key composition
compared input for input with the source's (query string, headers, cookies, path and compression); managed
default TTL values, cache-policy defaults and the invalidation quota each compared with the source's; the
rebuilt CDN configuration, cache policies and any edge code re-tested before cutover; the storage addressing
style in use recorded and consistent across every storage host; a bucket name containing a dot matched by its
certificate's subject-alternative-name coverage; the signed-URL signing scope and the credential-limited paths
inside the target's model; and object-storage consistency judged against the documented exceptions rather than
retried until visible; each certificate attached to a CDN distribution issued in the region that distribution
accepts; the DNS validation record left in place at issuance and still present so renewal continues.

**Pass threshold.** 100% of certificate checks pass with at least `[Min certificate validity]` remaining and
full hostname coverage; redirect and canonical rules match the source rule for rule; `robots.txt` and the
sitemap reachable with a success status and no accidental disallow; zero stale-content defects after an
invalidation or version change; each weight change fully effective within `[TTL propagation window]` at every
step; 100% of the cache-key inputs the source uses reproduced by the target's cache policy, with the managed
default TTL values, the cache-policy defaults and the invalidation quota each compared against the source and
every difference recorded as a finding with a decision; zero CDN or edge-code configurations accepted as a port,
each rebuilt configuration passing its re-test against the source's observed behaviour before the first traffic
step; 100% of storage hosts answering on the addressing style recorded in the plan, with a bucket name
containing a dot matched by its certificate; zero signed-URL requests rejected for signing scope or
credential-limited paths; zero reliance on retry-until-visible behaviour; every certificate attached to a CDN
distribution issued in the region that distribution accepts (`[CDN certificate region]`); every DNS validation
record still in place at the final traffic step, so renewal continues.

**Owner.** Cloud Platform Engineer, with the Application Owner for the SEO-critical content checks.

### 3.9 TA-9 Rollback

**Objective.** Prove the way back works, is fast enough to fit the permitted window, and correctly handles data
written to the target after cutover.

**Method.** Two rehearsals, both in P4 and both in the target non-production environment sized like production:
the first reverses the target before the dry run, and the second reverses a simulated traffic shift inside the
dry-run cutover. Each includes at least one write to the target after the shift, is timed end to end, and
reconciles those writes back into the source by the procedure stated at 3.11 — the named
write-consistency mechanism, the reconciliation report that evidences the reconciled record
count, and conflicts resolved by `[Conflict resolution rule]` under its named owner —
executed by the Data Engineer within the rehearsal the Cutover Manager owns.

**Example checks.** Reverse the DNS weights and confirm traffic returns to the source; confirm the target's
post-cutover writes are captured and reconciled or replayed; exercise the rollback trigger list against a
simulated threshold breach and time the decision; produce the reconciliation report; verify the critical
journeys still pass on the source after the reversal; verify the source's monitoring and alerting resumes at
full traffic.

**Pass threshold.** Both rehearsals, run consecutively in P4, complete within `[Max rollback time]`; zero data
loss across the reconciled writes, with the reconciled record count equal to the post-cutover write count in
each rehearsal; 100% of critical journeys pass on the source after the reversal; the rollback decision is
reached inside `[Decision timebox]` from the moment a trigger condition is observed.

**Owner.** Cutover Manager, with the SRE / Observability Engineer and the Data Engineer.

### 3.10 States every test area must handle

**Empty or absent.**

- **No source baseline exists.** The source baseline must be captured before testing starts; without it the performance comparison cannot proceed and TA-4 cannot report a result. Where a baseline is genuinely unavailable, the phase exit criteria record the gap with a named owner and an alternative reference for the threshold, rather than declaring the area passed.
- **A target component has no source equivalent.** The difference is recorded as a gap with a named owner, and the test for it asserts the component's own stated behaviour instead of a comparison. A gap with no owner is an exit-criteria failure for the phase it appears in.

**Failing.** Each failure state carries a stated decision and an action. Where a state below is also a rollback
trigger (section 5), precedence is fixed: the pause, halt or postpone path applies while no trigger has fired,
and once a trigger has fired the step or the cutover is reversed.

| Failure state | Decision and action |
|---------------|---------------------|
| A data validation mismatch — row count, checksum, hash, schema or orphan row — found before any traffic has moved | Halt traffic shifting; re-sync the affected scope; re-validate; resume only once the comparison is clean |
| A data validation mismatch found after a traffic step has begun | Reverse the step: this is a rollback trigger. The source is still live and in sync, so it resumes as the system of record; re-sync and re-validate before the step is attempted again |
| Error rate or latency beyond threshold during a traffic step, with no trigger fired | Pause the step inside its monitoring window, diagnose, and resume only once the breach has cleared; never advance to the next step on an unresolved breach |
| Error rate or latency beyond threshold that has met a trigger — `[Rollback error rate]` across two consecutive monitoring windows, or p95 above `[Rollback latency multiple]` of the source baseline for `[Rollback latency duration]` | Reverse the step or the cutover; the pause path may not be used to sit on a breach that has already met a trigger |
| Replication lag beyond `[Max replication lag at freeze]` at the freeze, before any traffic has moved | Postpone the cutover; do not advance, and do not accept a lagging target as the system of record. This is the postpone path and not a reversal, because no traffic has moved yet and there is no traffic step to reverse |
| Replication lag beyond `[Max replication lag at freeze]` after a traffic step has begun | Reverse the step, then re-sync and re-validate before retrying |
| A certificate or DNS failure — a certificate invalid, expired or missing for any host, or resolution failing for more than `[DNS failure threshold]` of probes | Roll the DNS weights back to the source, fix issuance or resolution, and re-enter the step from the beginning. Resolution failing at or below `[DNS failure threshold]` of probes is observed and diagnosed inside the monitoring window rather than reversed: the step never advances while the condition persists, and the window may not close inside its thresholds |
| A third party rejecting the target's traffic | Fix the allow-list or callback registration before the next traffic step; if the provider cannot be fixed inside `[Third-party fix window]`, reverse the step |

**Ordinary.** The step passes, the smoke test and every monitoring threshold are inside their limits, and the
sign-off for that step is recorded by the named owner before the next step begins. Nothing advances on a verbal
agreement.

### 3.11 Edge cases and their treatments

| Edge case | Stated treatment |
|-----------|------------------|
| Long-lived sessions and cookies split between the two environments during the traffic shift | Session state is held outside the environment that serves the request, or load-balancer stickiness is used, so a user routed to either environment keeps their session. The test asserts that a signed-in user stays signed in across a weight change and that cookie domain and `SameSite` settings behave on both hostnames. A forced sign-out during a traffic step is a functional defect, not accepted behaviour. |
| Cached DNS resolvers ignoring the lowered TTL | The TTL is lowered well before the window and the shift waits out the previous TTL. The source stays healthy for late-resolving clients, resolution is probed from multiple public resolvers, and the monitoring window is sized to the observed convergence rather than the planned one. Rollback does not depend on instant resolver convergence, so the way back stays reachable for at least the propagation window after a reversal. |
| Data written to both environments at once during the parallel run | One write path is designated authoritative and the other environment receives that change, or dual writes are reconciled by a named rule. The chosen mechanism is named and proven before P3 exits: the reconciliation report that proves it is produced and signed off by the Data Engineer, as the recorded evidence for the P3 exit criterion that the consistency mechanism is named and proven (2.3); conflicts are resolved by `[Conflict resolution rule]` with a named owner; and the reconciled record count must equal the dual-written record count. |
| Time-zone, character-encoding and collation differences between the source and target databases | Time-zone setting, character set and collation are compared explicitly for every database; the tests spot-check non-ASCII and multi-byte content, case sensitivity of lookups, sort order, and date and time rendering across a daylight-saving boundary and a time-zone offset. Any difference beyond the documented accepted list is treated as a data-integrity defect, not a cosmetic one; settings that are immutable after instance creation and settings that cannot be edited in the target's default parameter group are compared explicitly, any change that needs a restart is identified and applied before the comparison runs, table-name casing is compared, and the target's point-in-time recovery window is compared with the source's, with a shorter window recorded as a gap with a named owner rather than accepted silently. |
| Large uploads, signed URLs and CORS rules that depend on the storage domain | Signed-URL generation and consumption are tested from the target with lifetimes inside the provider's limit and after a key rotation; both environments cap a signed URL at seven days, so the differences that matter are the signing scope and the credential-limited paths inside the target's model; the largest supported upload is attempted end to end; CORS preflight and the origin allow-list are verified against the target storage hostname; the addressing style in use — virtual-hosted or path-style — is recorded and verified, and a bucket name containing a dot must be matched by the certificate presented for that name; object reads, writes and listings are judged against the target's documented consistency exceptions rather than retried until visible. A failure here halts the traffic step until it is fixed, because it is a failure of TA-3 as well as TA-8. |
| Scheduled jobs that run twice, or not at all, during the switch | One environment owns each schedule and the other environment's schedules are disabled for the window; job-run logs are compared to prove every job ran exactly once per schedule with no missed run; any job with an external side effect is checked for idempotency; a duplicate or missed run is treated as a functional defect at the cutover gate. |

## 4. Cutover and hypercare testing

### 4.1 The smoke set run at every traffic step

Every traffic step runs the same smoke set before its monitoring window opens. The smoke set is the short
cross-cutting set of the smoke level in 1.1:

- The homepage and a sample of public pages return a success status, with no certificate warning and no maintenance page, and with each page's analytics tags firing once against the target hostnames.
- Static assets load.
- One revenue-critical action completes end to end.
- The title, canonical and `robots` metadata are correct.
- `robots.txt` and the sitemap respond correctly.
- Authentication works: sign-in, session establishment and sign-out behave as on the source, and a signed-in session survives the step.
- One read-only administrative action succeeds, and a non-privileged role is denied an administrative action.
- The integration handshakes of TA-3 respond.

A failed smoke check stops the step: the step does not open its monitoring window and does not advance.
Where the failed check is a certificate or DNS failure or the integration handshake — a third party
rejecting the target's traffic — the matching row of 3.10 carries the decision and action: roll the DNS
weights back to the source and fix issuance or resolution before re-entering the step, or fix the
allow-list or callback registration before the next traffic step. Any other smoke failure — a static asset
that does not load, incorrect title, canonical or `robots` metadata, a `robots.txt` or sitemap failure,
broken authentication, or a failed read-only administrative action — has no row in that table and is
treated as a functional defect at the cutover gate, exactly as 3.11 treats a duplicate or missed scheduled
job; the smoke set is re-run only once TA-1 or TA-3 confirms the fix. Once the monitoring window is open,
a breach of the error-rate or latency thresholds is handled by the error-rate and latency rows of 3.10,
which pause the step inside the window, and any condition that has also met a rollback trigger (section 5)
is reversed rather than paused.

### 4.2 Traffic-shift steps and monitoring windows

Traffic moves in the steps **1%, 10%, 50% and 100%**. The target's DNS weight is staged at zero before the
first step and is raised only at that step, so no production traffic reaches the target until its smoke set has
passed at zero weight.

For each step: the smoke set runs, the step then holds for its monitoring window `[Step monitoring window]`,
and the step's sign-off is recorded by the named owner before the next step begins. A step that closes inside
every threshold advances; a step that breaches a threshold is handled by section 3.10, and a breach that has met
a rollback trigger reverses the step.

- **Cutover mechanics.** The weight mechanics, the TTL preparation and the mixed-traffic behaviour the steps rely on are stated once in 1.2 and are not restated here.
- **Certificate preparation.** Every certificate presented at a traffic step covers every hostname it serves with at least `[Min certificate validity]` remaining; a certificate attached to a CDN distribution is issued in the region that distribution accepts (`[CDN certificate region]`); and every DNS validation record is left in place at issuance, so issuance and renewal continue without a re-issue (TA-8).
- **Window sizing.** The monitoring window covers at least one representative peak-traffic period, and it is never shorter than the maximum rollback time `[Max rollback time]`, so a rollback remains possible inside the window.
- **Writes during the shift.** Data written while traffic is shifting is accounted for and reconciled, which is a P5 exit criterion. The measurable criterion is TA-9's equal-record-count rule (3.9), extended to the shift window: the reconciled record count equals the record count written to the target during the shift, every write made during the shift appears in the reconciliation, and any conflicting write is resolved by `[Conflict resolution rule]`. TA-2 produces that reconciliation — data integrity is the area P5 runs for writes during the shift — and its owner, the Data Engineer, records the written and reconciled counts the Cutover Manager signs the P5 exit against.

### 4.3 Hypercare

Hypercare runs for `[Hypercare duration]` after the final traffic step. Its exit is by signal rather than by
date, and every signal is measurable:

- Issue volumes are at or below the pre-cutover baseline.
- No highest-severity incident has been open longer than `[Max P1 open age]`, and no major-severity incident longer than `[Max P2 open age]`.
- Latency and error rate have stayed inside threshold (TA-4) for the duration of hypercare.
- The live production failover test of TA-6 has been completed, which is the controlled fault injection in production approved for hypercare.
- The final reconciliation of writes is complete and the source no longer receives production writes.

Until those exit criteria are met, the source stays live and in sync and the way back stays available: TA-9
keeps the reversal ready and timed throughout hypercare, and section 5's triggers remain in force for the whole
period, not only for the traffic steps.

### 4.4 Downtime and the permitted change window

Production downtime during cutover is assumed to be minimal or none. The permitted change window — its
duration, its time of day and any freeze it sits inside — is `[Permitted cutover window]`. It is confirmed
during P4, checked as a go/no-go entry in section 5 before the first production traffic step, and carried as an
assumption in section 8: the rollback triggers and the monitoring windows are sized against it, so it cannot be
left unset.


## 5. Go/no-go checklist and rollback triggers

### 5.1 Go/no-go checklist for cutover

Every entry below is measurable, and every entry must be green before the first production traffic step. The
decision is recorded as a signed checklist line by the Cutover Manager; nothing advances on a verbal agreement.

1. All P4 exit criteria are signed off, and the dry-run cutover in the production-like target environment has completed successfully.
2. The source baseline is captured, with no critical journey missing a baseline measurement.
3. Data integrity is clean: zero count, checksum and hash mismatches, zero orphan rows, and replication lag inside `[Max replication lag at freeze]` and stable for `[Stability window]`.
4. The target smoke set (4.1) passes at zero DNS weight.
5. Observability is complete: 100% of the alert inventory provisioned and proven to fire, and retention explicitly set on every log destination.
6. Security is clean: zero unresolved critical or high findings, secrets held in the managed store with one rotation proven, and encryption at rest and in transit verified.
7. Resilience targets are met: measured RTO and RPO at or below `[RTO target]` and `[RPO target]` in rehearsal, in two consecutive runs.
8. Integration is complete: 100% of allow-list and callback re-registrations confirmed in writing by the Third-Party Integration Owner.
9. Both rollback rehearsals are complete within `[Max rollback time]`, each including post-cutover writes.
10. The TTL has been lowered and the previous TTL has expired, and the target's DNS weight is staged at zero.
11. The permitted cutover window `[Permitted cutover window]` is confirmed, the change freeze is in effect, and the support rota is staffed for the hypercare duration `[Hypercare duration]`.

### 5.2 Rollback triggers

Any one of the following reverses the traffic step or the cutover. Each is measurable, and each takes
precedence over the pause-and-recover path in 3.10: a trigger that has fired is reversed, never paused on.

1. A critical journey's error rate above `[Rollback error rate]` across two consecutive monitoring windows.
2. p95 latency above `[Rollback latency multiple]` of the source baseline sustained for `[Rollback latency duration]`.
3. Any data integrity mismatch discovered after the shift — a count, checksum, hash or orphan-row difference.
4. Replication lag beyond `[Max replication lag at freeze]` at the point of freeze, once a traffic step has begun.
5. A certificate invalid, expired or missing for any host, or resolution failing for more than `[DNS failure threshold]` of probes.
6. A security incident attributable to the migration — an unintended exposure, a credential leak, or exploitation of a rule that should have blocked it.
7. A third party rejecting the target's traffic and not fixable inside `[Third-party fix window]`.
8. A fault that cannot be diagnosed and given a resolution path inside `[Decision timebox]`, in which case the default is rollback rather than continue.

### 5.3 The rehearsed rollback

The way back is rehearsed twice, consecutively, in P4 and before the first production traffic shift. The first
rehearsal reverses the target before the dry run; the second reverses a simulated traffic shift inside the
dry-run cutover. Each rehearsal covers data written to the target after the shift, each is timed end to end
against the maximum rollback time `[Max rollback time]`, and each reconciles those writes with a reconciled
record count equal to the post-cutover write count (TA-9).

If either rehearsal exceeds `[Max rollback time]`, the shortfall is an exit-criteria failure for P4 and the
cutover does not proceed. Because the source is the system of record throughout, a rollback returns users to an
environment that has stayed live and in sync, and the reversal itself is a DNS weight change rather than a
rebuild.


## 6. Roles, environments, risks and mitigations

### 6.1 Roles and responsibilities

Role names only: no team or person name appears anywhere in this plan.

| Role | Responsibility | Environments |
|------|----------------|--------------|
| Test Lead | Owns the plan, runs the gates, keeps the assumption list current, and signs phase exits with the owning role | All three environments, plus the external control surfaces when a gate is signed |
| QA / Test Engineer | Executes functional, regression and integration checks, and records every result against its threshold | Source production (read-only), target non-production, target production during smoke steps |
| Application Owner | Confirms the critical journeys and admin functions; owns TA-1 and the hypercare exit, delegating TA-1 execution to the QA engineers | Source production, target non-production, target production |
| Cloud Platform Engineer | Owns TA-8 and the target platform build; environment readiness and decommission readiness | Source production (read-only), target non-production, target production |
| Data Engineer | Owns TA-2: replication, validation, reconciliation and restore tests | Source production (read-only), target non-production, target production |
| Performance Engineer | Owns TA-4: source baseline capture and target comparison | Source production (approved load window), target non-production, target production (passive observation during hypercare) |
| Security Engineer | Owns TA-5: policy, network, TLS, edge firewall, secrets and encryption verification | Source production (read-only configuration inventory), target non-production, target production |
| SRE / Observability Engineer | Owns TA-6 and TA-7: failover tests, alert proofs and on-call readiness | Source production (read-only monitoring inventory), target non-production, target production |
| Third-Party Integration Owner | Owns TA-3: allow-list, callback and webhook re-registration with each provider | Target non-production, target production, plus each provider's own console as an external control surface |
| Cutover Manager | Owns TA-9, the traffic steps, the go/no-go decision and the rollback decision | Target non-production, target production, source production, plus the DNS control plane as an external control surface |
| Programme Stakeholder | Signs the gates against the recorded exit criteria and the assumption list | None: review only |

### 6.2 Environments

Three environments are used. They are defined here and referenced by every phase and test area.

| Environment | Definition | Used for |
|-------------|------------|----------|
| Source production | The existing GCP environment serving live traffic | Baseline capture, smoke and functional comparison, and the rollback target of record |
| Target non-production, sized like production | An AWS environment with production-equivalent instance sizes, database class and storage, serving no live traffic | Target environment validation, data migration validation at volume, the pre-cutover rehearsal and the dry-run cutover |
| Target production | The AWS environment that will serve live traffic after cutover | Cutover, the hypercare checks, and the post-cutover test types |

Two external control surfaces sit outside these three environments and are recorded as control surfaces, never
as environments: the **DNS control plane**, where record weights and TTLs are set, and **each third-party
provider's own console**, where allow-lists, callbacks, webhooks and redirect URLs are registered. Tests reach
them from the environments above, and configuration changes to them are made by their named owners — the
Cutover Manager for DNS, the Third-Party Integration Owner for each provider.

Producing this document creates none of these environments: the plan assumes the existing source production
environment, a target non-production environment sized like production, and the target production environment,
and it provisions nothing.

### 6.3 External dependencies the plan relies on

Each dependency is a precondition the migration team satisfies later, and each has an owner role.

| Dependency | Why the plan needs it | Owner role |
|------------|----------------------|------------|
| Target cloud account with rights to create and configure the target estate | The target non-production and production environments, and the resources each test area inspects | Cloud Platform Engineer |
| Network connectivity between the two clouds for data sync and replication | The parallel run, data migration validation and reconciliation | Cloud Platform Engineer, Data Engineer |
| DNS control of the website's zones, including the ability to set record weights and TTLs | The traffic-shift steps, the cutover and the rollback | Cutover Manager |
| Certificate issuance authority, including control of the DNS validation record and the ability to leave that record in place | Certificate validity and renewal on the target: renewal depends on the validation record being left in place, and a certificate attached to a CDN distribution must be issued in a region that distribution accepts | Cloud Platform Engineer |
| Access to each third party whose allow-lists, callback, webhook or redirect URLs must change | Integration re-registration and its verification | Third-Party Integration Owner |
| Read access to source production monitoring and configuration | Baseline capture; the source redirect, canonical, cache and certificate reference rule set; the security configuration inventory; the monitoring inventory; and post-cutover comparison | Test Lead, Security Engineer, SRE / Observability Engineer, Cloud Platform Engineer |
| An approved load-testing window against source production | The performance baseline in TA-4 | Performance Engineer |
| Approval to run controlled fault injection in production during hypercare | The production failover test in TA-6 | SRE / Observability Engineer |
| Backup and restore tooling, and the authority to restore into a clean environment | The measured RTO and RPO in TA-6 | Data Engineer |

Two boundaries hold across all of them. Producing this plan creates no infrastructure and accesses no cloud
account, so every dependency above is a precondition the migration team satisfies. And no dependency may be
satisfied by putting live customer personal data into a non-production environment (section 1.3).

### 6.4 Risk register and mitigations

| Risk | Mitigation | Test area |
|------|------------|-----------|
| Silent data loss or corruption during migration | Layered validation with counts, checksums, schema parity and orphan checks before any traffic moves | TA-2 |
| Functional regression on a critical user journey after the move | Baseline the journeys on the source, then replay them on the target before the first traffic step | TA-1 |
| Scheduled jobs running twice or not at all during the switch | Single-owner schedule, run-log comparison, idempotency checks | TA-1 |
| A third party rejecting the target's traffic or calling a stale callback | Re-register allow-lists and callback URLs before the step that depends on them, then test them from the target | TA-3 |
| Performance regression or capacity shortfall under peak load | Baseline-then-compare on identical scenarios, including a peak and a spike profile | TA-4 |
| Autoscaling too slow or unstable to absorb a spike | Observe scale-out timing and oscillation during the spike profile, and set limits on both | TA-4 |
| Over-permissive identity, an exposed network path, weak TLS, a missing edge rule, plaintext secrets or unencrypted data | Least-privilege review, segmentation checks, handshake tests, rule-parity review, secret-store and encryption verification, and scanning | TA-5 |
| Edge traffic reaching the target before it is ready | Stage the target's DNS weight at zero, require a passing smoke set, and add it only at the first step | TA-8 |
| A certificate or DNS failure at cutover, or stale resolver caching | Lower the TTL ahead of the window, validate certificates per host, probe from multiple resolvers, keep the reversal reachable | TA-8 |
| SEO and URL regressions after the host change | Redirect, canonical, sitemap and `robots.txt` checks after every traffic step | TA-8 |
| A zone, instance or database failure taking the site down on the target | Multi-zone and failover tests with measured RTO and RPO before cutover | TA-6 |
| Backups that cannot be restored inside the recovery objectives | Restore into a clean environment and measure the achieved time rather than the documented one | TA-6 |
| Operating blind because dashboards, alerts or logs do not exist on the target | Inventory parity plus a fault-injection proof that each alert fires and routes | TA-7 |
| Rollback failing, exceeding the permitted window, or losing data written to the target | Rehearse the reversal in production-like conditions, time it, and reconcile post-cutover writes | TA-9 |
| Diverging data while both environments accept writes | Name and prove the consistency mechanism, and reconcile with equal record counts | TA-2, TA-9 |

## 7. Traceability

### 7.1 Test areas, the risks they address and the environments they run in

| Test area | Risks addressed | Environments |
|-----------|-----------------|--------------|
| TA-1 Functional and regression | Functional regression; duplicate or missed scheduled jobs | Source production (read-only), target non-production, target production |
| TA-2 Data integrity | Silent data loss or corruption; diverging data during the parallel run | Source production (read-only), target non-production, target production |
| TA-3 Integration | A third party rejecting the target's traffic; stale callbacks and allow-lists | Target non-production, target production |
| TA-4 Performance and load | Performance regression; capacity shortfall; slow or unstable autoscaling | Source production (approved load window), target non-production, target production (passive observation during hypercare) |
| TA-5 Security | Over-permissive identity; exposed network path; weak TLS; missing edge rule; plaintext secrets; unencrypted data | Source production (read-only configuration inventory), target non-production, target production |
| TA-6 Resilience and failover | Zone, instance or database failure; unrestorable backups | Target non-production, target production |
| TA-7 Observability | Operating blind without dashboards, alerts or logs on the target | Source production (read-only monitoring inventory), target non-production, target production |
| TA-8 DNS, TLS and CDN | Traffic arriving before the target is ready; certificate or DNS failure; SEO and URL regression | Source production (read-only), target non-production, target production |
| TA-9 Rollback | Rollback failure; over-long rollback; lost post-cutover writes; diverging data | Target non-production, target production, source production |

Every test area above reaches at least one risk, and every risk in the register at 6.4 reaches at least one test
area: neither table contains an orphan. Which test types run in which phase is stated once, in the test-types
table at 2.8, and is not repeated here.

### 7.2 Service-mapping coverage

Every row of the mapping table at 1.4 is covered by at least one test area.

| Mapping row | Covered by |
|-------------|-----------|
| Compute | TA-1, TA-4, TA-6 |
| Relational database | TA-2, TA-4, TA-6 |
| Object storage | TA-2, TA-8 |
| Load balancing and CDN | TA-4, TA-6, TA-8 |
| DNS and certificates | TA-8 |
| Identity and secrets | TA-5 |
| Observability | TA-7 |
| Edge security | TA-5, TA-8 |

Where a mapping row turns out not to apply to the website, it is recorded as a gap with a named owner rather
than silently dropped: until a gap carries an owner and a plan, the exit criteria of the phase it appears in
are not met. The two rows flagged in 1.4 — edge security, and identity and secrets — are the likeliest to
change what a test area must prove, and either can turn a comparison test into a test of the component's own
stated behaviour.

## 8. Assumptions to confirm

### 8.1 Assumptions

This final section lists the assumptions the plan rests on, not facts it establishes. Each is stated as an
assumption, with the reason it matters, and the plan is revised as the programme confirms them and as each
phase's exit criteria are signed off.

| Assumption | Why it matters |
|------------|----------------|
| The website is a dynamic web application with a relational database and object storage, running on the services in the mapping table | The mapping drives the test areas; if the site is static or has no database, those rows become recorded gaps |
| Every mapping row is assumed until confirmed, including the two flagged rows — edge security, and identity and secrets | An inapplicable row changes what a test area must prove, and a missing counterpart is a gap with an owner |
| The cutover is a parallel run with progressive weighted DNS traffic shifting, not a big-bang switch | The whole phase sequence and the rollback design rest on it |
| The traffic steps are 1%, 10%, 50% and 100%, each with a monitoring window | The smoke tests and the go/no-go decision attach to the steps |
| The migration is like-for-like, with no application changes | A feature change would invalidate the baseline comparison |
| Downtime is minimal or none, and the permitted change window is `[Permitted cutover window]` | The rollback trigger set and the go/no-go timing depend on the window |
| The hypercare duration is `[Hypercare duration]` | The hypercare exit criteria are measured against it |
| The maximum rollback time is `[Max rollback time]` | It bounds both rollback rehearsals and the decision timebox |
| Every other value left as a bracketed placeholder is to be agreed by the programme before the phase that first uses it | Each is a threshold or a window that a test area, a gate or a trigger depends on; none may be left unset, and none may be replaced by a figure invented without the Application Owner's agreement |
| The TTL is lowered to `[TTL value]` at least 24-48 hours before the window opens | Resolver caching makes the shift effective only after the previous TTL expires |
| The source's current behaviour is the reference for functional, security and performance comparison | Without a captured baseline, TA-4 cannot report a result |
| An anonymised production copy is available and permitted for use in the test environments | Data validation, regression and load comparison all depend on realistic data |
| The environments are the existing source production, a target non-production sized like production, and the target production environment | The phases and their owners are assigned to these three |
| The deliverable is one Markdown document, high level only, containing no test cases, scripts or automation | It is the boundary between this plan and the runbook and the test assets |
| The migration runbook exists and is owned elsewhere | The plan references it and does not replace it |
| Non-website workloads are migrated separately | Their absence from this plan is deliberate, not an omission |

### 8.2 The bracketed placeholders to be agreed

Each of the thirty placeholders below is a value the programme has not yet supplied. Each appears in the
plan only where it is needed, none is replaced by an invented figure, and each is agreed before the phase that
first depends on it.

- **Performance, capacity and data thresholds:** `[Agreed latency tolerance]`, `[Agreed error-rate tolerance]`, `[Max sustained utilisation]`, `[Max scaling events]`, `[Max scale-out time]`, `[Max replication lag at freeze]`, `[Stability window]`, `[Failover error tolerance]`, `[RTO target]`, `[RPO target]`.
- **Cutover, DNS and TLS mechanics:** `[TTL value]`, `[TTL propagation window]`, `[Step monitoring window]`, `[Permitted cutover window]`, `[DNS failure threshold]`, `[Min certificate validity]`, `[Minimum TLS policy]`, `[CDN certificate region]`.
- **Operational windows and rollback thresholds** (including the incident-severity ages `[Max P1 open age]` and `[Max P2 open age]`): `[Alert delivery target]`, `[Max P1 open age]`, `[Max P2 open age]`, `[Rollback error rate]`, `[Rollback latency multiple]`, `[Rollback latency duration]`, `[Max rollback time]`, `[Decision timebox]`, `[Third-party fix window]`, `[Conflict resolution rule]`, `[Decommission observation window]`, `[Hypercare duration]`.

Until a placeholder is agreed, the test area, gate or trigger that depends on it cannot be reported as passed.

