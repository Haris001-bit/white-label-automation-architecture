# Deep dive: vision estimation and input gating

Detail on [§6](../README.md#6-vision-estimation) and [§7](../README.md#7-the-determinism-boundary) of the README.

The estimation rubric itself is the client's product and is not described here. Everything below is about the machinery around it, which is where the engineering difficulty actually sat.

---

## The failure that defines the problem

A vision estimator's characteristic failure is not misjudging a good photograph. Models are reasonably good at that.

It's producing a confident estimate from a photograph that could not possibly support one — a picture too dark to read, framed so tightly there's no sense of scale, or of something else entirely. Ask a vision model to estimate from that and it will. Fluently, with a specific number, and with no indication anything was wrong.

Downstream, nothing can detect this. The number is in a plausible range, the email renders correctly, the customer receives it. The error surfaces when someone physically arrives and finds reality unrelated to the estimate.

**So the first real component is not the estimator. It's the refusal.**

---

## Gating

Gates run before estimation, cheapest first, and each has a distinct rejection message.

| # | Gate | Checks | Cost |
|---|---|---|---|
| 1 | Technical | Resolution, file size, corruption, aspect ratio | Free |
| 2 | Exposure/blur | Brightness histogram, Laplacian variance | Free |
| 3 | Subject presence | Is the thing actually in frame? | 1 vision call |
| 4 | Framing/scale | Fully visible? Scale reference present? | same call |
| 5 | Category match | Does the subject match what the form claims? | same call |
| 6 | Completeness | Are required views present? | Logic |

Gates 1–2 are classical image processing and cost nothing — run them first and reject the large share of bad submissions before spending an inference call. Gates 3–5 fold into a single structured vision call that returns assessments rather than an estimate.

### Rejection messages are a conversion surface

A gate that says `Unable to process image` loses the lead. The customer doesn't know what to fix, so they don't.

A gate that says *"We can't see the whole subject — could you step back and include something for scale, like a car or a person?"* gets a usable photo back a high proportion of the time.

**Every rejection path is a re-engagement opportunity, and writing those messages carefully is worth more than a marginal improvement in model accuracy.** This is product work living inside an engineering component, and it's routinely left as a stub.

### Gate results are logged, always

Every gate decision is recorded with its reason. This gives you:

- The distribution of *why* submissions fail — which drives what you fix, in the product and in the customer-facing instructions
- A tuning surface. Gates that are too strict show up as rejections on images a human judges fine
- Evidence when a customer disputes a rejection

Without gate logging, "why did we reject this?" is unanswerable, and gate thresholds get tuned by argument rather than by data.

---

## Separating observation from computation

The core structural decision, and the one that generalises furthest.

The vision model does **not** produce a price. It produces observations:

```json
{
  "subject_present": true,
  "fully_visible": true,
  "scale_reference": "vehicle",
  "category": "<classification>",
  "measurements": { "<dimension>": "<bucket>" },
  "access_obstacles": ["<observed>"],
  "confidence": 0.86,
  "notes": "partial occlusion, left third"
}
```

A deterministic engine consumes that structure and produces the number.

### Why this is not merely tidier

**Auditability.** An estimate decomposes into observations plus rules. When one is wrong, you can tell *which* was wrong — the model misread the image, or the rule was miscalibrated. Those have completely different fixes, and without the split you cannot distinguish them.

**Testability.** The rules engine takes known input to known output. Real regression tests, run in CI, no inference cost. Prompt-based pricing can only be evaluated statistically, over a sample, at cost, with noise.

**Changeability.** A rate change is a config edit. Seven tenants adjust independently, safely, without touching a prompt or re-validating a model's arithmetic.

**Tenant isolation.** Same observations, different rate tables, different outputs. If pricing lived in the prompt, per-tenant pricing would mean seven prompts, and seven prompts is seven things that drift.

**The generalisation, which is the most transferable thing in this repository:**

> Use models to convert unstructured input into structured observations. Use code to convert structured observations into decisions. The boundary between them is where correctness, testability and auditability all live.

Most production LLM failures I've seen are a violation of that boundary — a model asked to compute, decide, or enforce, rather than to perceive and describe.

---

## Ranges and confidence

Output is a range with a confidence level, never a point estimate.

### The information argument

Photographs underdetermine the answer. Some inputs aren't recoverable from an image at all — conditions not visible from the available angle, access constraints outside frame, material properties you cannot see. A point estimate asserts precision the input doesn't contain.

### The trust argument, which matters more

A point estimate that later changes is experienced as a bait-and-switch, even when the revision is entirely legitimate. A range that resolves to a value inside itself is experienced as *accurate* — the customer's expectation was set correctly and then met.

Same underlying uncertainty. Completely different customer experience, determined entirely by output format.

**Output format is a trust decision, not a presentation detail.**

### Width should track confidence

A fixed-width range is a lie in both directions — too wide on clear inputs, too narrow on ambiguous ones. Range width derives from the confidence the observations carry:

| Observation confidence | Range | Message |
|---|---|---|
| High | Narrow | Standard estimate |
| Medium | Wider | Estimate with a note on what would firm it up |
| Low | **No estimate** | Offer a site visit |

That last row is the same principle as gating, applied one stage later: **"I can't answer this" must remain available all the way to the end of the pipeline**, not only at the entrance. An estimator that always estimates will always estimate badly on its worst inputs.

---

## Prompt and model discipline

Notes that apply regardless of what the rubric contains.

**Structured output, schema-validated.** The vision call returns JSON validated against a schema. A parse failure routes to human handling — never a partial or coerced record passed downstream.

**Pin the model version.** Vision model behaviour shifts between versions, and a shift in an estimation pipeline is a shift in customer-facing prices. Pin explicitly, test before upgrading, and treat an upgrade as a change requiring re-validation against a held-out set.

**Keep a regression set of real submissions with known-correct assessments.** Run it before any prompt or model change. Without it, "did that change help?" is unanswerable and every adjustment is a gamble.

**Temperature at or near zero.** This is a perception task, not a creative one. Variance across identical inputs is pure defect.

**Log the full model input and output for every estimate.** When a customer disputes one, the image, the observations, the rules that fired, and the final number all need to be reconstructible. Without it, a dispute is unresolvable and you concede by default.

**Never let the image carry instructions.** Text visible in a submitted photograph is untrusted input reaching a model. Treat the image as data, keep the instruction region structurally separate, and don't let observed text alter behaviour.

---

## What I'd do differently

**Build gating before estimation.** Estimation came first because it's the interesting part; gates were added as bad inputs surfaced in production. That ordering meant a period of confident wrong estimates reaching real customers, which is the exact failure the system most needed to avoid.

**Log gate decisions from the first version.** Gate thresholds were tuned by intuition for too long because the data to tune them properly wasn't being captured — and it cost nothing to capture.

**Build the regression set on day one.** Every prompt change before it existed was an unmeasured gamble, and the set could have been assembled from traffic that was already flowing past.
