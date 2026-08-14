# White-Label Automation Architecture

Engineering notes from operating one automation product across **seven customer deployments** — an AI vision estimation pipeline cloned per client, each with its own rate card, its own sending identity, and its own destinations, all maintained by one person.

The recurring question in this shape of work is deceptively simple: **when a product serves many customers, do you build one instance that knows about all of them, or many instances that each know about one?** Both answers are defensible. Picking the wrong one for your constraints is expensive in a way that takes months to become visible.

Design notes and reasoning only. No client name, no source code, no credentials, no rate cards, no customer data, and no disclosure of the estimation rubric itself — that logic is the client's product and it stays theirs.

**Deep dives:** [Clone-per-tenant tradeoffs](docs/01-clone-per-tenant.md) · [Vision estimation and input gating](docs/02-vision-estimation.md)

---

## Contents

1. [Problem shape](#1-problem-shape)
2. [One instance or many](#2-one-instance-or-many)
3. [Configuration as data](#3-configuration-as-data)
4. [Credential portability](#4-credential-portability)
5. [Identity: shared infrastructure, separate presentation](#5-identity-shared-infrastructure-separate-presentation)
6. [Vision estimation](#6-vision-estimation)
7. [The determinism boundary](#7-the-determinism-boundary)
8. [Building a connector that doesn't exist](#8-building-a-connector-that-doesnt-exist)
9. [Voice follow-up](#9-voice-follow-up)
10. [Failure modes](#10-failure-modes)
11. [What I'd do differently](#11-what-id-do-differently)

---

## 1. Problem shape

A service business sells a lead-capture and estimation product to other businesses in the same trade. Each customer gets what looks like their own system:

- Their branding, their sending address, their phone number on outbound messages
- Their own pricing — same estimation logic, materially different numbers
- Their own CRM account, their own notification destinations
- Their own customers, who must never see evidence that anyone else exists

Behind it, one codebase, one maintainer, and a deployment cadence measured in days rather than quarters.

**The tension:** customers experience isolation, the maintainer needs leverage. Every architectural decision below is a trade between those two.

---

## 2. One instance or many

The two viable shapes, stated honestly, because the usual writeup pretends there's only one right answer.

### Single multi-tenant instance

One workflow. Tenant identity is a parameter. Config is looked up per request.

| | |
|---|---|
| ✅ | One place to fix a bug. Every customer gets the fix at once |
| ✅ | Consistent behaviour by construction |
| ✅ | Adding a customer is a row, not a deployment |
| ❌ | One bad deploy breaks **every** customer simultaneously |
| ❌ | Tenant leakage is a live risk on every code path |
| ❌ | Per-customer divergence fights the architecture |

### Clone per tenant

One instance per customer, each with its own configuration.

| | |
|---|---|
| ✅ | Blast radius is exactly one customer |
| ✅ | Per-customer divergence is trivial — it's a different instance |
| ✅ | No tenant-leakage class of bug. There is no shared query path |
| ✅ | A customer can be paused, migrated, or offboarded independently |
| ❌ | A fix must be propagated N times. Drift is the default outcome |
| ❌ | Onboarding is a deployment |
| ❌ | Observability fragments across N instances |

### What actually decided it

Not elegance. **Blast radius and divergence rate.**

These deployments were customer-facing and revenue-carrying: an estimate reaching a homeowner with the wrong price is a real commercial event for that business. A single-instance bug at seven customers is seven simultaneous incidents and seven angry phone calls to one maintainer.

And the customers genuinely diverged. Different rate structures, different notification preferences, different follow-up cadences. In a single instance those become conditional branches, and the conditional-branch count is the thing that eventually makes multi-tenant systems unmaintainable — every branch multiplies the state space that any future change has to be correct across.

**The heuristic I'd now use:**

> Clone when tenant count is low, divergence is high, and per-tenant blast radius matters more than propagation cost. Consolidate when tenant count is high, divergence is low, and you have the engineering capacity to make isolation a first-class property rather than a hope.

Seven customers with real divergence and one maintainer sits clearly on the clone side. Seventy would not — the propagation cost crosses over well before that, and §11 covers what I'd build to handle the crossover.

Full analysis: [docs/01-clone-per-tenant.md](docs/01-clone-per-tenant.md)

---

## 3. Configuration as data

The decision that makes cloning survivable: **clones differ only in data, never in logic.**

```
instance = shared_logic + tenant_config
```

Every clone runs identical structure. What varies is a config surface:

| Config | Examples |
|---|---|
| Pricing inputs | Rate table, multipliers, minimums, surcharges |
| Identity | Sending address, display name, reply-to, branding assets |
| Destinations | Notification recipients, CRM account, alert channels |
| Behaviour flags | Follow-up on/off, voice enabled, review requests enabled |
| Thresholds | Input quality floors, auto-send vs review, escalation triggers |

### The rule that holds it together

**If a customer needs different behaviour, it becomes a config field — never a forked branch in that clone.**

The moment one clone's logic diverges structurally, propagation stops working: the next fix can't be applied blindly, someone has to remember which clone is special, and that knowledge exists in one person's head until it doesn't.

This costs real discipline. A one-line change in a single clone is always faster today than adding a config field to all seven. Taking that shortcut twice produces a fleet nobody can safely update.

**Config divergence is manageable. Logic divergence compounds.**

---

## 4. Credential portability

The most-improvised part of the system, and the one with the clearest lesson.

### The problem

The orchestration platform stores third-party credentials in its own internal credential store, referenced by ID. That works fine for one instance. Across clones it produces a specific pain: credentials are bound to the platform's storage, so cloning a workflow doesn't clone its access, migrating an instance means re-authorising everything by hand, and rotating a token means touching every clone individually.

For OAuth tokens that refresh on a schedule, this becomes a recurring manual task multiplied by the number of clones.

### What was built

An external token store the clones read from and write back to at refresh time, decoupling credential lifecycle from the platform's internal store. Clones became genuinely portable — a new deployment pointed at the store and worked, and a rotation happened in one place.

### Being honest about it

It solved a real operational problem and I'd make the same call again under the same constraints — one maintainer, seven instances, a platform whose credential model didn't fit the deployment model.

But it should be named for what it is: **a workaround for a platform limitation, carrying real security tradeoffs.** Tokens living outside a purpose-built secret store means the access controls are whatever that store offers, the audit trail is whatever it happens to log, and encryption at rest is a property you inherit rather than choose.

Done again with more headroom: a proper secrets manager with per-tenant scoping, short-lived tokens, and a documented rotation path. The engineering lesson generalises past the specific hack —

**When a platform's credential model doesn't fit your deployment model, that mismatch will surface as recurring manual work. Solve it deliberately and early, because the improvised version is the one that ends up load-bearing.**

---

## 5. Identity: shared infrastructure, separate presentation

Customers must appear independent. That does not require independent infrastructure everywhere, and knowing where the line falls saves substantial cost.

| Layer | Shared? | Why |
|---|---|---|
| Messaging transport | Shared | Recipients see content and sender identity, not your account structure |
| Sending phone number | Shared, in this build | Per-tenant numbers cost more and add provisioning; revisit when volume or compliance demands |
| Email sender | **Per tenant** | The address is the brand. Visible on every message |
| CRM account | **Per tenant** | Customer records must never commingle. Non-negotiable |
| Notification destinations | **Per tenant** | Obviously |

The reasoning: **share what the end customer cannot observe; separate what they can, and separate absolutely anything that holds their data.**

Worth being clear about the shared sending number, since it's the compromise in the list. It was the right call for the volume and the budget, and it has real limits — a deliverability or reputation problem on that number affects every deployment at once, and per-tenant number provisioning is the correct answer as volume grows. It's a deliberate trade with a known trigger for revisiting, not an oversight.

---

## 6. Vision estimation

A customer submits photographs and a short form. A vision model assesses the images against a structured rubric and returns an estimate, delivered by email and SMS.

The rubric itself — what's measured, how it's bucketed, how factors combine — is the client's product and is not described here. **What generalises is everything around it**, and in practice that's where the engineering difficulty was.

### Input gating is most of the work

A vision estimator's dominant failure mode is not misjudging a good photograph. It's confidently estimating from a photograph it should have refused.

Gates that ran before any estimation:

| Gate | Rejects |
|---|---|
| Usability | Too dark, too blurred, too small, corrupted |
| Subject presence | The thing being estimated isn't in frame |
| Framing / scale | No reference for size; too close or too far |
| Category | Subject is a different class than the form claims |
| Completeness | Required angles or views missing |

Each gate returns a **specific, actionable message** — "we need a photo showing the whole subject with something for scale" — not a generic failure. A rejection that tells the customer exactly what to re-send converts; a generic error loses the lead.

**Refusing to estimate is a feature, and it needs to be built as deliberately as estimating.** A model asked to price an unusable photo will price it, fluently, and the number will be wrong in a way nobody can detect downstream.

### Ranges, not points

Output is a range with an explicit confidence, never a single number. Two reasons, and the second matters more:

Photographs genuinely underdetermine the answer — some information isn't recoverable from an image, and a point estimate claims precision the input doesn't support.

And a range sets the right expectation for the human visit that follows. A point estimate that later moves is experienced as a bait-and-switch; a range that resolves inside itself is experienced as accurate. **The output format is a trust decision, not a formatting one.**

Full detail: [docs/02-vision-estimation.md](docs/02-vision-estimation.md)

---

## 7. The determinism boundary

The same line that governs any money-touching LLM system, and it's worth stating as a rule because it's the most common place production LLM work goes wrong:

| Transformation | Handler |
|---|---|
| Unstructured → structured | Model. This is what they're for |
| Structured → structured | **Deterministic code. Always** |
| Structured → prose | Model, within a template |

The vision model's job is **observation**, not arithmetic: it reports what it sees as structured measurements and classifications. A deterministic engine turns those observations into a price.

### Why the line is not negotiable

A model asked to compute a price will return a plausible one. Plausible is useless — the number is either the customer's actual rate or a mistake they may be held to, and there is no way to distinguish a correct number from a confident one by inspection.

Keeping arithmetic deterministic also buys three things that matter operationally:

- **Auditability.** Every estimate decomposes into which rules fired with what inputs
- **Testability.** Known inputs, known outputs, real regression tests
- **Changeability.** A rate change is a config edit, not a prompt rewrite and a re-evaluation

That last one is what makes the fleet maintainable at all. Seven customers adjusting rates independently is trivial against a rules engine and genuinely dangerous against a prompt.

---

## 8. Building a connector that doesn't exist

The CRM in this stack had no off-the-shelf integration on the automation platform. It was built by hand — OAuth2 authorisation flow, token refresh, and a GraphQL client against the vendor's API.

Notes for anyone facing the same:

**Token refresh is the whole problem.** The initial authorisation is a day. Refresh handling — concurrency when several runs refresh at once, clock skew, revocation, the provider rotating the refresh token itself on use — is where the time goes and where the 3am failures come from.

**GraphQL is a real advantage here.** Requesting exactly the needed fields in one round trip beats REST's over-fetch-and-discard, and the schema is introspectable, which means you can discover the API rather than reverse-engineer it from documentation that lags the implementation.

**Write it once, expose it as an internal interface.** The connector became a shared component every clone called, not seven copies of the same auth logic. Auth code is precisely what you do not want duplicated across a fleet — it's the code most likely to need an urgent, uniform change.

**Version-pin and monitor for schema drift.** A vendor deprecating a field is a silent break; you find out when a downstream field is quietly empty. A daily introspection diff is cheap insurance.

---

## 9. Voice follow-up

Automated outbound calls handle post-service follow-up and review requests.

Design constraints that turned out to be the whole job:

**Timing is the dominant variable.** Call too early and the work isn't finished; too late and the customer has moved on. This is a scheduling problem with business rules, not an AI problem, and it deserves more design attention than the conversation does.

**Voicemail must be detected and handled distinctly.** An agent delivering an interactive script to an answering machine is a bad customer experience that the customer keeps as a recording.

**Every call needs an unambiguous exit to a human**, and taking it must be trivially easy. An agent that makes escalation difficult converts a neutral interaction into a complaint.

**Respect calling windows per recipient timezone**, and treat this as a hard constraint enforced in code. Regulatory in many jurisdictions, and independently just correct.

**Cap attempts per contact, globally.** Retry logic that seems reasonable per-workflow becomes harassment when several workflows can each call the same person. The cap belongs at the contact level, not the workflow level — this is the same blast-radius reasoning as everything else here.

---

## 10. Failure modes

| Failure | Symptom | Mitigation |
|---|---|---|
| Clone drift | One customer behaves differently after a fix | Config-only divergence; version stamp per clone; periodic diff |
| Partial fleet update | Fix applied to some clones | Deployment checklist with per-clone confirmation; version audit |
| Token expiry mid-run | Silent integration failure on one clone | Centralised refresh; alert on refresh failure, not just on request failure |
| Cross-tenant config bleed | Customer A's rates on customer B's estimate | Config loaded once at entry, immutable thereafter; assert tenant on write |
| Vision estimate from unusable input | Confident wrong price to a real customer | Pre-estimation gates; refusal path with actionable guidance |
| Shared number reputation damage | Deliverability drops for all deployments | Volume monitoring; per-tenant numbers as the growth path |
| Duplicate outbound contact | Customer contacted repeatedly | Contact-level attempt cap across all workflows |
| CRM schema change | Field silently empty | Scheduled introspection diff; alert on drift |

**Clone drift is the defining risk of this architecture** and it's worth treating as such rather than as an operational annoyance. Every clone carries a version stamp, and a fleet-wide audit answers "which clones are behind" as a query rather than a memory exercise. Without that, drift accumulates invisibly until a customer reports behaviour nobody can reproduce.

---

## 11. What I'd do differently

**Build the fleet manager before the third clone.** Cloning by hand is fine at two and untenable at seven. A single tool that lists every clone with its version and config diff, and pushes an update across all of them, was needed well before it existed. The build cost is small; the accumulated manual cost is not.

**Version-stamp clones from day one.** Retrofitting version tracking onto a fleet already in production means auditing each one by hand to establish where they started.

**Use a real secrets manager.** The token store solved the problem, and it should have been a purpose-built secret store with per-tenant scoping from the beginning. See §4 — I stand by the decision under the constraints, not as a pattern to copy.

**Design the config schema first, formally.** It grew organically, which meant fields that should have been config existed as constants in a couple of clones before anyone noticed. A declared, validated schema with a default per field would have prevented the class entirely.

**Instrument per-clone from the start.** Fragmented observability was the real cost of cloning, and it's the one I underestimated. Structured logs with a tenant tag flowing to one place gives you the isolation benefit of clones without the visibility loss — and it's much cheaper to build in than to add across seven live deployments.

---

## Stack

n8n · vision and text LLM inference · deterministic rules engine · voice AI with telephony integration · hand-built OAuth2 + GraphQL CRM connector · Supabase · Google Sheets · scheduled and webhook-triggered orchestration

---

*Engineering notes only, published with the client's permission. Contains no source code, credentials, API keys, phone numbers, endpoints, rate cards, customer data, commercial terms, client-identifying information, or the estimation rubric itself.*
