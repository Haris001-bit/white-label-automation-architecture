# Deep dive: clone-per-tenant tradeoffs

Detail on [§2](../README.md#2-one-instance-or-many) and [§3](../README.md#3-configuration-as-data) of the README.

---

## The decision, framed properly

Most writing on multi-tenancy assumes the shared-instance answer and treats cloning as what you do before you know better. That's wrong often enough to be worth arguing against.

The real question is which of two costs you'd rather pay:

| Architecture | You pay in |
|---|---|
| Shared instance | **Isolation engineering** — every code path must be correct for every tenant, forever |
| Clone per tenant | **Propagation** — every change must reach N instances, forever |

Both are recurring costs. Neither disappears with cleverness. Choosing well means knowing which one your constraints make cheaper — and the constraints that matter are team size, tenant count, divergence rate, and blast-radius tolerance.

---

## When cloning wins

**Low tenant count.** Propagation cost scales linearly with N. At 5–10 it's a checklist. At 100 it's a full-time job and the architecture has failed.

**High divergence.** If tenants genuinely need different behaviour — not different values, different *behaviour* — a shared instance accumulates conditionals. Every conditional multiplies the state space every future change must be correct across, and that growth is what eventually makes shared-instance systems unchangeable.

**Blast radius dominates.** When one bad deploy hitting all tenants simultaneously is unacceptable — revenue-carrying, customer-facing, or a small team with no capacity for a multi-tenant incident — isolation by construction is worth real propagation cost.

**Small team, no platform engineering capacity.** This is the underrated one. Shared multi-tenancy done *safely* needs isolation as a first-class engineering property: enforced tenant predicates, cross-tenant tests, leak assertions, per-tenant rate limiting. That's a real investment. A team without capacity for it doesn't get "simpler" by choosing a shared instance — they get a shared instance with latent tenant-leakage bugs, which is strictly worse than clones.

**Per-tenant lifecycle independence.** Pause one, migrate one, offboard one, run one on an older version during a migration. Trivial with clones; a feature you must build with a shared instance.

---

## When cloning loses

**Tenant count grows past what a person can hold.** Propagation crosses from checklist to job somewhere around 10–20 without tooling.

**Divergence is actually low.** If tenants differ only in values, a shared instance with a config lookup is strictly simpler and you should take it.

**Fixes must be simultaneous.** A security patch that must land everywhere at once is a strong argument against cloning — partial fleet update is a real state, and during it some tenants are vulnerable.

**Aggregate analytics are a product requirement.** Cross-tenant reporting against N separate instances is a data-engineering project. Against one shared store it's a query.

---

## The hybrid, and why it's usually right at scale

Shared logic, cloned execution:

```
        ┌──────────────────────────┐
        │   shared component lib   │   auth, connectors, estimation
        │   (versioned, single     │   engine, notification layer
        │    source of truth)      │
        └────────────┬─────────────┘
                     │  referenced, not copied
     ┌───────────────┼───────────────┐
     ▼               ▼               ▼
 ┌────────┐     ┌────────┐     ┌────────┐
 │Tenant A│     │Tenant B│     │Tenant C│
 │ config │     │ config │     │ config │
 └────────┘     └────────┘     └────────┘
```

Each tenant keeps an isolated execution context and its own config, but shared logic lives in one versioned place that instances *reference* rather than *contain*.

This gets most of the isolation benefit with much of the propagation benefit, and it's the shape I'd build toward now. The catch is that it requires the platform to support shared, versioned components — many low-code orchestration tools don't, or support it weakly, which is exactly how you end up with full copies instead.

**When the platform can't express "shared logic, separate execution," you are choosing between full duplication and full sharing, and the tradeoff table above is the real decision.**

---

## Making clones survivable

If you clone, these stop being nice-to-haves.

### Version stamping

Every clone records which version of the shared logic it's running. Non-negotiable. Without it, "which clones have the fix" is answered by memory, and memory is wrong.

```
tenant_a  v1.4.2   updated 2026-08-10
tenant_b  v1.4.2   updated 2026-08-10
tenant_c  v1.3.9   updated 2026-07-22   ← behind, 2 versions
```

That table is the single highest-value artefact in a cloned fleet. It converts drift from an invisible accumulating risk into a visible work queue.

### A propagation checklist with confirmation per clone

Applying a fix to seven instances means seven confirmations, recorded. "I think I did them all" is how one clone silently stays broken for a month.

### Config schema, declared and validated

Config is the only thing allowed to vary, so it needs to be a real schema — every field declared, typed, defaulted, validated at load.

Organic config growth produces the failure where a value that should be per-tenant exists as a constant in some clones and a field in others. That's logic divergence wearing config's clothing, and it defeats the entire discipline.

### Centralised observability

Clone isolation fragments visibility by default, and this is the cost people underestimate.

The fix is cheap if done early: structured logs and metrics from every clone, tagged with tenant, flowing to one destination. You keep isolation of *execution* while regaining a single view of *behaviour*. Retrofitting this across live deployments is much harder than starting with it.

### A fleet manager, before you think you need it

One tool that lists every clone with version and config, diffs any clone against the canonical version, and pushes an update across all of them.

Build it at three clones. At seven it's overdue; the manual cost by then has already exceeded what building it would have taken.

---

## The migration path

Cloning is not permanent. The realistic sequence:

| Phase | Tenants | Shape |
|---|---|---|
| Early | 1–3 | Clone. Manual propagation is genuinely fine |
| Growth | 4–10 | Clone + fleet manager + version stamping |
| Scale | 10–30 | Hybrid: shared versioned logic, per-tenant config and execution |
| Platform | 30+ | Shared instance with isolation as a first-class engineered property |

Each transition is triggered by propagation cost exceeding isolation cost, and the trigger is measurable: **when time spent propagating changes exceeds what a tenant-leak-proof shared instance would cost to build and maintain, move.**

The mistake is jumping to the last row at three tenants because it's the architecture that sounds most mature. Premature multi-tenancy buys isolation engineering you don't need yet, at the price of the blast-radius protection you do.
