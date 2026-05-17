# MCP Manifest Generator — Source of Truth

**Version 1.0 · May 2026 · Maintained by James Holm**

This is the canonical spec for the MCP Manifest Generator. Any AI agent or human contributor rebuilding, extending, or modifying this product should treat this document as authoritative. If the code and this doc disagree, the doc wins — and the code gets fixed to match it. If the doc is wrong, the doc gets fixed first, before the code.

---

## Mission — Make MCP manifests something a marketing ops practitioner could trust, not just something a developer could write.

The MCP Manifest Generator turns any API description (OpenAPI, Postman, repo, docs URL) into a production-ready MCP server. It exists because hand-authored manifests are slow, error-prone, and produce tool definitions Claude can't reliably select against. The product replaces that workflow with a five-step wizard that ends in a signed, tested, safety-scoped manifest.

Every output the product produces must ladder up to three things:

1. **Selection accuracy.** Will Claude pick the right tool the first time, given the user's prompt?
2. **Safety scoping.** Are destructive operations explicitly fenced behind approval, by default?
3. **Confidence at ship time.** Does the user know the manifest works *before* it touches production?

If a feature, copy change, or design choice doesn't ladder up to one of these three, flag it and stop. This is the product's RICE — selection accuracy, safety, confidence. Everything else is decoration.

---

## Core Principles

- **The manifest is the deliverable.** Everything in the wizard is in service of producing a manifest the user can ship. UI polish, animations, copy — none of it matters if the output manifest isn't useful. Reviewers should be able to read `manifest.json` and understand what was built.
- **Defaults bias toward safety.** When the classifier is uncertain, it flags. When a description rewrite is ambiguous, the original is kept. When a destructive override is requested, it requires a typed confirmation. The user has to opt *into* risk, never opt out of safety.
- **Mocks are a feature, not a shortcut.** The optimizer scoring, harness results, and registry push are simulated. This is intentional. It makes the demo deterministic, offline-capable, and free of API-key risk. The architecture allows real API integration as a v2.1 add; the absence of it in v2 is a design choice, not a gap.
- **One sample is the headline.** Stripe is the canonical demo path. Every screen has been tuned to render well with the Stripe spec. Other APIs work — Petstore, billing, anything OpenAPI-shaped — but Stripe is what we show.
- **No backend.** The entire product is a single HTML file. State lives in memory. This is a hard architectural constraint, not a v1 limitation. A backend would require auth, secrets, persistence, hosting — and none of that earns the user anything they can't get by re-running the wizard.
- **Iterate this document.** When the product changes, this doc changes first. Don't ship code that this doc doesn't predict.

---

## 1. Plan Before Building

- For any change that touches more than one of the five components, write the plan in this doc first.
- If something goes sideways mid-task — Stripe mock data drifts from real Stripe semantics, a new MCP spec field appears, a screen feels wrong — **STOP and re-plan.** Don't push through.
- Plans should name: the user, the problem, the success criteria, the affected components, and what the kill criteria for the change are.

## 2. Component Strategy

- Each of the five components below owns one lane. Do not let them drift outside their scope.
- Components communicate through shared state (`state.manifest`, `state.diffs`, `state.safety`, `state.harnessResults`) and nothing else.
- A component that needs data from another component reads it from state. Components do not call each other directly.
- The wizard shell is the orchestrator. It controls step transitions and gates ("can't reach Step 4 until Step 1 has a parsed manifest").

---

## 3. The Five Required Components

Every iteration of this product must engage all five components below. Each has a single, narrow focus. The components run in strict sequence — the user cannot reach Step N until Step N-1 has produced its output — though edits in earlier steps invalidate later state and rerun is allowed.

### 3.1 Parser (Step 1)

**Only focus:** Reading an API description and normalizing it into a tool list.

- Accepts OpenAPI 3.x (YAML or JSON) and Postman Collection v2.x.
- Extracts every operation: name, description, HTTP method, path, parameters, request body schema.
- Produces a normalized `tools` array. Each tool has: `name`, `description`, `inputSchema` (JSON Schema, type "object"), and internal-only `_method` and `_path` for downstream components.
- Resolves `$ref` references one level deep. Deeper resolution is out of scope for v2.
- **Does NOT:** rewrite descriptions, classify safety, generate test results, or modify the input. Hands off a clean tool list and nothing else.

**Output:** A `state.manifest` object: `{ name, version, description, tools: [...] }`.

**Kill criteria (stop and reconsider if):**
- The parser silently succeeds on malformed input. Bad input must throw with a human-readable message.
- More than 5% of real-world OpenAPI specs from a representative sample fail to parse with no useful error.
- The output tool list contains duplicate names, empty descriptions, or schemas with no properties when properties exist in the source.
- Operation IDs are silently regenerated when the source has them. We use what the spec provides.

### 3.2 Optimizer (Step 2)

**Only focus:** Rewriting tool descriptions for AI-readability and showing the user the diff.

- For Stripe (the headline demo), uses pre-computed rewrites from `STRIPE_OPTIMIZER_DATA`. Every rewrite has: `original`, `optimized`, `baselineAccuracy` (0–100), `optimizedAccuracy` (0–100), and `recommended` (bool).
- For any other spec, synthesizes a plausible mock rewrite via `synthesizeOptimizerEntry()`. The synthesized version expands the original description with a "use when..." clause and assigns a random-but-believable accuracy delta (baseline 60–78%, optimized 75–96%, with at least +15pp lift).
- Renders a two-column diff per tool: original on left (red-tinted), optimized on right (green-tinted), accuracy delta above ("Selection accuracy: X% → Y%").
- Accept / Keep original buttons per tool. Bulk Accept All / Keep All Originals at the top.
- **Does NOT:** classify destructive operations, run tests, or modify the parsed schemas. Only touches `description` strings.

**Output:** Mutates `state.diffs[toolName].accepted` (bool). On ship, accepted rewrites replace original descriptions; rejected rewrites leave originals intact.

**Kill criteria (stop and reconsider if):**
- An optimized description loses information the original had (e.g., drops required parameters, omits deprecation notes).
- The accuracy delta is shown without a sample size context. Currently rendered as "on 25 held-out prompts" — this attribution must stay.
- A rewrite is accepted without the user explicitly seeing the diff. Bulk-accept is allowed; silent-accept is not.
- The synthesized fallback for non-Stripe specs ever shows >+40pp lift. That's unbelievable and breaks trust.

### 3.3 Safety Classifier (Step 3)

**Only focus:** Categorizing each tool's risk profile and surfacing rationale.

- For Stripe, uses pre-computed classifications from `STRIPE_SAFETY_DATA`. Every classification has: `category` (one of `destructive`, `write`, `read`), `rationale` (one sentence), `example` (a plausible natural-language prompt that would invoke the tool).
- For any other spec, falls back to `classifyTool()`: matches destructive verbs (delete, remove, send, transfer, charge, publish, cancel, modify, update, revoke) against the tool name and description, *and* uses HTTP method as a signal (DELETE = destructive even if name doesn't match; GET = not destructive unless name explicitly says so).
- Renders tools grouped by category, color-coded: destructive (red), write (amber), read (green), unknown (gray).
- Every destructive tool shows: HTTP method badge, name, rationale, example invocation, and an "Allow without approval" override link.
- Override flow requires a `confirm()` dialog with friction-by-design language ("Type carefully — this disables the default safety scope"). After override, the row visibly states "Override active" with an option to re-enable.
- **Does NOT:** rewrite descriptions, run tests, or change the manifest's tool list. Only annotates risk.

**Output:** Mutates `state.safety[toolName].category` and `state.safety[toolName].override` (bool). On ship, every tool gets an `annotations` block: `{ requiresApproval, destructiveHint, readOnlyHint }`.

**Kill criteria (stop and reconsider if):**
- A destructive operation is classified as read-only. Recall on the destructive class is non-negotiable; false positives on read-only are acceptable, false negatives on destructive are not.
- The override flow is reachable without an explicit confirmation. One-click destructive override is a security regression.
- Rationale strings ever exceed two sentences. They must be skimmable.
- The example invocation is a generic placeholder ("example call") rather than a plausible natural-language prompt.

### 3.4 Test Harness (Step 4)

**Only focus:** Simulating 50 prompts per tool and surfacing tools that need re-review.

- For Stripe, uses pre-computed pass rates in `generateHarnessResults()` — each tool has a target rate (0.74–1.00) and 50 individual pass/fail outcomes generated from that rate via `Math.random()`. The randomness is intentional: identical demos should produce subtly different heatmaps, like a real eval would.
- For any other spec, synthesizes a pass rate of 0.75–0.98 per tool via `synthesizeHarnessForTool()`.
- Simulates a 1.8-second "running" delay before results render. Long enough to feel like work is happening; short enough not to slow the demo.
- Renders four summary stats: overall pass rate, total prompts run, tools ≥80%, tools flagged (below 80%).
- Renders a per-tool heatmap: 50 cells per tool, green for pass, red for fail, with the percentage on the right. Cells are 9×18px and hover-scale subtly.
- Any tool below 80% gets a flagged callout below the heatmap: tool name, pass rate, and a one-sentence diagnosis ("ambiguous parameter selection").
- **Does NOT:** rerun the optimizer, change classifications, or block shipping. The harness flags; it doesn't gate.

**Output:** Sets `state.harnessResults` — an object mapping tool names to `{ rate, results: [50 strings] }`.

**Kill criteria (stop and reconsider if):**
- The harness is presented as if real API calls happened. Copy must reference "simulated prompts" or "simulated harness" — never imply a live eval ran.
- A tool flagged below 80% blocks the user from continuing. The user makes the call to ship or fix.
- Pass rates are too uniformly high (>95% for every tool). The harness must look like it actually finds problems — for Stripe, `createCharge` and `updateSubscription` should hit the flagged range often.
- The "Run harness" button can be triggered without a parsed manifest. The state gate must hold.

### 3.5 Exporter (Step 5)

**Only focus:** Producing the final downloadable artifacts and the summary card.

- Builds the final manifest via `buildFinalManifest()`. For each tool: takes the optimized description if the diff was accepted, otherwise the original. Attaches an `annotations` block from the safety state.
- Renders three tabs: Summary (four stat cards mirroring Step 4's layout), `manifest.json` (the MCP manifest), `server.json` (the same content with the MCP server schema reference at the top).
- Three actions: Download `manifest.json`, Download `server.json`, Push to registry (mocked — changes the subtitle to a success state).
- Summary card shows: tool count, accepted rewrites, count of tools requiring approval, harness pass rate.
- **Does NOT:** make network calls, sign anything cryptographically, or modify state. Pure read-from-state and serialize.

**Output:** Two downloadable JSON files matching MCP server schema, plus an in-page summary view.

**Kill criteria (stop and reconsider if):**
- The downloaded `manifest.json` doesn't match what was shown in the JSON tab.
- The exported manifest is missing the `annotations` block on any tool, or has annotations that contradict the user's overrides.
- The "Push to registry" button is presented as if it actually pushed anywhere. Copy must remain mock-honest.
- The Summary tab uses different stat formatting than the Test step. Visual consistency is a hard requirement across these two screens.

---

## 4. The Wizard Shell

The shell is the orchestrator. It owns:

- **Step state.** `state.currentStep` (1–5). Mutated only via `goToStep(n)`.
- **Hash routing.** Each step has its own URL: `/#step-1` through `/#step-5`. The hash updates on every `goToStep()` call. On `hashchange`, the wizard restores the requested step *if* the manifest exists (Step 1 is always reachable; Steps 2–5 require `state.manifest`).
- **Gating.** Calling `goToStep(n > 1)` without `state.manifest` shows an error and refuses. This is the only flow control the wizard enforces.
- **Stepper UI.** Five pills with step number + label. Active step is highlighted in white; completed steps show a filled circle; future steps are dim. Clicking any pill calls `goToStep()` (gated by the rule above).
- **Animation.** Step transitions use a 0.4s fade-in. Smooth-scroll to the stepper on transition. No other animations.

The shell **does not** know what any individual component does. It just transitions between them and gates access.

---

## 5. Verification Before Done

Nothing ships from this codebase without:

- **Parser check:** The embedded Stripe sample (`STRIPE_SAMPLE_SPEC`) parses to exactly 15 tools, every operationId matches a key in `STRIPE_OPTIMIZER_DATA` and `STRIPE_SAFETY_DATA`. If you change the sample spec, you change both lookup tables in the same commit.
- **Demo check:** The full Stripe → Parse → Accept All → Run Harness → Download path works without console errors.
- **Mobile check:** The wizard is usable at 380px wide. The diff view collapses to single-column. The stepper hides labels and shows numbers only.
- **Public-repo check:** No console.logs, no TODOs, no debugger statements, no API keys, no internal company references. Run `grep -in -E "todo|fixme|console\.log|debugger|api_key" index.html` before any commit.

If any of the four checks fails, the change is not done. Fix and re-verify.

---

## 6. Design System

The product's visual identity is non-negotiable and must survive every iteration. It's the single biggest reason this looks like a portfolio piece instead of a class project.

**Colors (CSS variables in `:root`):**
- `--black: #080808` — background
- `--white: #f0ece4` — warm off-white, primary text
- `--dim: rgba(240, 236, 228, 0.38)` — secondary text, labels
- `--dimmer: rgba(240, 236, 228, 0.1)` — borders, dividers
- `--accent: #5b9cf5` — interactive accent (blue)
- `--green: #6ee7a0` — success, read-only, pass states
- `--red: #f87171` — destructive, failure
- `--yellow: #fbbf24` — warning, idempotent write

**Typography:**
- Headlines: Playfair Display, 400 weight, italic where emphasized
- Body: DM Sans, 300 weight (light)
- Mono: SF Mono / Fira Code stack, used only in code blocks and tool names
- Labels: 0.62rem, letterspacing 0.18em, uppercase, color `--dim`

**Layout:**
- Max content width: 980px
- Card padding: 2rem on desktop, 1.5rem on mobile
- Card border: 1px solid `--dimmer`, border-radius 6px, subtle warm fill `rgba(240,236,228,0.015)`
- Buttons: 0.65rem × 1.4rem padding, border-radius 4px, no shadows, no gradients

**Don'ts:**
- No emojis in UI strings. Use Unicode glyphs (`&middot;`, `&rarr;`, `&mdash;`) instead.
- No shadows on buttons or cards. Depth comes from the warm-fill technique, not drop shadows.
- No gradients except the noise overlay on the body.
- No icon libraries. The product uses zero icon images. Color, type, and layout do all the work.
- No bold weight in body text. If something needs emphasis, italicize or change color.

---

## 7. Data Model

The five components communicate exclusively through this state shape:

```js
state = {
  currentStep: 1,                  // 1..5
  manifest: {                      // null until Step 1 parses successfully
    name: 'stripe',
    version: '1.0.0',
    description: 'MCP manifest for stripe',
    tools: [
      {
        name: 'listCharges',
        description: 'Returns a list of charges...',
        inputSchema: { type: 'object', properties: {...}, required: [...] },
        _method: 'get',
        _path: '/v1/charges'
      },
      // ...
    ]
  },
  diffs: {                         // keyed by tool.name
    listCharges: {
      accepted: true,
      optimizer: {
        original: '...',
        optimized: '...',
        baselineAccuracy: 68,      // 0..100
        optimizedAccuracy: 94,     // 0..100
        recommended: true
      }
    }
  },
  safety: {                        // keyed by tool.name
    listCharges: {
      category: 'read',            // 'destructive' | 'write' | 'read' | 'unknown'
      rationale: '...',
      example: '...',
      override: false              // only meaningful for destructive
    }
  },
  harnessResults: {                // null until Step 4 runs
    listCharges: {
      rate: 0.98,                  // target pass rate, 0..1
      results: ['pass', 'pass', 'fail', ...] // exactly 50 entries
    }
  },
  isStripeDemo: false              // true when the Stripe sample was loaded; gates pre-computed mock data
}
```

This shape is the contract between components. Adding a field is a non-breaking change; renaming or removing a field requires updating every component that reads it.

---

## 8. Out of Scope for v2

The following are intentionally not built. Each is a deliberate trade-off, not an oversight.

- **No real API integration.** No live calls to Claude, no live calls to the underlying API the manifest targets. Everything is mocked. Reason: deterministic demo, zero credential risk, no API-cost surprises. v2.1 candidate.
- **No backend.** No auth, no user accounts, no persistence. Reason: would require infrastructure not earned by v2's value prop. v3 candidate if the product needs registry hosting.
- **No file-based persistence.** Refreshing the page clears state. Reason: localStorage is forbidden in the deployment environment, and adding a backend just for persistence isn't worth it. The wizard is fast enough to re-run.
- **No real registry.** The "Push to registry" button is a visual mock. Reason: hosting a public manifest registry has security and moderation implications well beyond a class project.
- **No GraphQL or SOAP parsing.** OpenAPI 3.x and Postman v2.x only. Reason: REST covers the demo audience; GraphQL is a 2-3 day add by itself.
- **No multi-level `$ref` resolution.** One level deep only. Reason: real-world OpenAPI specs rarely chain refs more than once for the demo cases.
- **No automated screenshot or video output.** The "demo video" is recorded manually. Reason: not a generic value to users, and out of scope for a 2-week build.
- **No accessibility audit beyond keyboard-navigable buttons.** Reason: known limitation; would need a real WCAG pass for v2.1.
- **No analytics or telemetry.** Reason: no value to the user, and adds compliance complexity.

If a feature being asked about is on this list, the answer is "intentionally not in v2." Don't sneak it in without a plan-mode discussion.

---

## 9. Demand Elegance (Balanced)

- For any non-trivial output (a new component, a redesigned screen, a new mock dataset): pause and ask "is there a more elegant way to express this?"
- If a section feels padded or generic: strip it and rewrite from the insight. The product's design language is restraint; the code's design language should match.
- Skip this for simple, obvious work. Don't over-engineer a button color change.
- Challenge your own work before presenting it.

## 10. Autonomous Iteration

- When given a critique that's mechanical (a typo, a color tweak, a copy fix): just fix it. Don't ask for hand-holding.
- For substantive critiques (a different demo flow, a new component): propose the revised version *before* implementing, then implement after approval.

---

## Output Standards (All Components)

Every artifact produced by this product must:

- Match the design system exactly. No inline styles that contradict the CSS variables.
- Be reachable from the wizard's hash routes. If a screen exists but doesn't have a route, it's not done.
- Survive being viewed on mobile (380px wide) without horizontal scroll.
- Pass the public-repo check before committing.

## On Completion

When a substantial change ships:

1. Verify against all four checks in Section 5.
2. Update this document if any of Sections 1–8 are now wrong.
3. Commit with a message that names what changed and what didn't.

---

## Living Document Protocol

This file improves as the product evolves.

- After every meaningful change: review which sections helped, which got ignored, which needed more detail.
- Version and date every revision at the top of this file.
- When in doubt: tighten, don't pad. The best version of this doc is the one a contributor (human or AI) actually reads end-to-end before touching code.
