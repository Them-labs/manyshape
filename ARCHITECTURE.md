# Manyshape - technical architecture

Companion to [README.md](README.md). This document specifies the four subsystems: the **capability contract** (what vendors ship), the **runtime** (what users run), the **agent loop** (how interfaces get made and remade), and the **registry** (how it all distributes and upgrades).

```
┌─────────────────────────── user's device ───────────────────────────┐
│  Manyshape Runtime                                                       │
│  ┌───────────────┐  ┌──────────────────┐  ┌──────────────────────┐  │
│  │ Interface      │  │ Activation gate   │  │ Sandbox              │  │
│  │ agent          │─▶│ types · policies  │─▶│ surfaces + workflows │  │
│  │ (intent →      │  │ · conformance     │  │ (WASM/iframe, no     │  │
│  │  surface)      │  │   tests           │  │  ambient authority)  │  │
│  └───────┬───────┘  └──────────────────┘  └──────────┬───────────┘  │
│          │ reads                                      │ calls        │
│  ┌───────▼───────────────────────┐          capability handles       │
│  │ Intent spec store (user-owned)│                    │              │
│  └───────────────────────────────┘                    │              │
└───────────────────────────────────────────────────────┼──────────────┘
                          ▲ signed contracts            │ authz enforced
                 ┌────────┴────────┐          ┌─────────▼──────────┐
                 │ Manyshape Registry  │          │ Authority plane     │
                 │ contracts ·     │          │ (vendor-hosted      │
                 │ interface packs │          │  services + data)   │
                 └─────────────────┘          └────────────────────┘
```

---

## 1. The capability contract

A contract is a signed, semver'd package. Directory layout:

```
mail-app@2.1.0/
├── contract.json          # manifest: id, version, planes, signatures
├── schemas/               # data model (JSON Schema / TypeSpec)
│   ├── thread.json
│   └── message.json
├── capabilities/          # typed actions - the ONLY way surfaces touch data
│   ├── archive.cap.json
│   ├── send.cap.json
│   └── query.cap.json
├── policies/              # invariants any surface/workflow must satisfy
│   ├── destructive-confirm.policy.ts
│   └── send-requires-gesture.policy.ts
├── events/                # subscribable event streams (thread.created, …)
├── surface/               # reference UI, shipped AS SOURCE (remix substrate)
│   ├── tokens.json        # design tokens: color, type, spacing, motion
│   ├── components/        # idiomatic components the agent remixes
│   └── screens/           # default screens (the out-of-box product)
├── conformance/           # executable acceptance criteria (the quality floor)
│   ├── reachability.test.ts
│   ├── safety.test.ts
│   └── a11y.test.ts
└── docs/                  # agent-readable semantics (llms.txt-style)
```

### 1.1 Capabilities

A capability is a typed, effectful function with declared semantics. Example:

```jsonc
// capabilities/archive.cap.json
{
  "id": "mail.archive",
  "input": { "threadId": "schema:thread#id" },
  "output": { "ok": "boolean" },
  "effects": ["mutates:thread.state"],
  "authz": "caller.uid == thread.owner",     // enforced in the authority plane
  "risk": "reversible",                       // reversible | destructive | external
  "idempotent": true,
  "rate": { "per_minute": 120 },
  "docs": "Moves a thread out of the inbox. Inverse: mail.unarchive."
}
```

Key properties:
- **Authz is evaluated server-side** in the authority plane on every call. The contract copy is documentation; the authoritative check never leaves the vendor.
- **`risk` and `effects` drive policy.** A policy like `destructive-confirm` binds to `risk: destructive` capabilities generically, so it survives contract evolution.
- **Capabilities are the unit of permission.** A surface or workflow declares which capabilities it needs; the runtime injects only those handles. Nothing else is reachable - no fetch, no DOM outside its mount, no other apps.

### 1.2 Policies

Policies are pure functions over (surface manifest, static analysis of surface, capability set, runtime context) → allow/deny/require-change. Three binding times:
- **Activation-time** (most): "any surface using `risk:destructive` capabilities must render a confirmation affordance" - checked by static analysis + conformance test.
- **Call-time**: "`mail.send` requires a user gesture within the last 500 ms" - enforced by the runtime bridge.
- **Data-time**: "PHI fields render only through the vetted `<Redactable>` component" - enforced by taint-tracking schema fields through the component tree at activation.

Vendors write policies in TypeScript against a small, analyzable API; the runtime executes them in its own sandbox (policies are vendor code running on user machines - they get no more ambient authority than surfaces do).

### 1.3 Conformance tests

The vendor's answer to "infinite interfaces, whose quality bar?" - executable specs run headlessly against any candidate surface before activation:

```ts
// conformance/reachability.test.ts
conformance("compose is always reachable", async (surface) => {
  const path = await surface.shortestPathTo(cap("mail.compose"));
  expect(path.interactions).toBeLessThanOrEqual(2);
});
```

The harness drives the surface in a headless renderer using the accessibility tree (which doubles as the a11y check). Failing surfaces are never activated; the agent gets the failure as feedback and iterates. Vendors ship these once; every user-generated interface inherits the bar.

### 1.4 What ships as source vs. stays closed

| Plane | Ships as | Why |
|---|---|---|
| Authority (services, data, business logic) | Nothing - stays vendor-hosted | Moat + enforcement must be trusted |
| Contract (schemas, caps, policies, tests) | Signed metadata + sandboxed code | Agents need semantics, not implementation |
| Surface (components, screens, tokens) | Source | Agents remix idiomatic examples; generation-from-nothing is the failure mode |

This is why the only source required is the layer users were always going to regenerate anyway - not the vendor's backend or business logic.

---

## 2. The runtime

Web-first host (a PWA-like shell; later Electron/mobile shells embedding the same core). Responsibilities:

### 2.1 Sandbox
- Surfaces run in isolated realms: an iframe/ShadowRealm per app for DOM surfaces; **WASM components (WASI preview 2 model)** for workflows/middleware. No ambient capabilities - no network, no storage, no timers beyond a budget - everything arrives as injected handles.
- The **capability bridge** is the only door: a typed RPC boundary that (a) validates inputs against schemas, (b) enforces call-time policies, (c) attaches auth and forwards to the authority plane, (d) logs to the audit trail.
- Cross-app workflows get handles from multiple contracts, each independently policy-checked; the workflow itself remains sandboxed with the union of nothing.

### 2.2 Activation gate
Every surface/workflow the agent produces passes, in order: schema/type check → static policy analysis → conformance suite in the headless renderer → signature + provenance recording. Only then is it swapped in. Failure at any step returns structured feedback to the agent. Worst case at runtime: an activated surface throws → the runtime catches, falls back to the **reference surface** (which always ships and always passes), and files the trace for the agent to repair. The user is never stranded on a broken screen.

### 2.3 State
- **Intent specs** (§3.2) and generated surface source live in user-owned storage (local + synced), encrypted; the vendor never needs them.
- App data lives where it always did - the authority plane. Manyshape adds a client cache keyed by schema for offline reads, invalidated by contract events.
- **Audit log:** every agent modification is a commit - diffable, revertible, attributable. "What changed and why" is a first-class screen.

---

## 3. The agent loop

### 3.1 From utterance to interface

1. **Intent capture:** user talks to the runtime agent ("make this a kanban of things awaiting my reply"). The agent reads contract docs, schemas, capabilities, and the current intent spec.
2. **Intent spec update:** the request is normalized into the spec (below) - *before* any code is written. The spec, not the code, is the durable artifact.
3. **Surface synthesis:** the agent **remixes, not generates**: it starts from the vendor's reference components and screens (idiomatic source, correct token usage, correct capability wiring) and restructures. Blank-page generation is the fallback, not the default.
4. **Gate + iterate:** activation gate failures feed back; the agent iterates until pass or bounded-retry, then presents a live preview diff ("here's your new inbox - the old one is one tap away").

### 3.2 The intent spec

A structured, human-readable document - the user's "interface DNA":

```yaml
app: mail-app
contract_range: ">=2.0 <3.0"
identity: "inbox as task list; zero-inbox workflow; keyboard-first"
structure:
  - view: kanban
    columns: {group_by: awaiting_reply_state}
    item: thread-card {show: [sender, age, one_line_summary]}
behaviors:
  - on: thread.archived    then: treat-as-completed
  - workflow: auto-snooze  {when: "unanswered > 48h", cap: mail.snooze}
suppress: [marketing-folder, promotions-tab]
style: {density: compact, motion: minimal}
provenance:
  - {date: 2026-07-04, utterance: "I live in my inbox as a task list…"}
```

Properties that matter:
- **Contract-relative, not code-relative.** It references schemas and capabilities by ID, so it survives any rewrite of the surface code.
- **Portable.** Published to the registry (minus provenance) it becomes an **interface pack** others apply; their agent re-derives against *their* contract version, locale, and device.
- **Composable.** A user's global spec ("compact, keyboard-first, dark") layers under per-app specs - design decisions that cross app boundaries, which no per-app settings page could ever offer.

### 3.3 Middleware on the fly

Workflows are how surfaces change behavior, not just layout. The agent writes a small typed function; the runtime compiles it to a WASM component:

```ts
// generated workflow: declared caps = [mail.query, mail.snooze]
export async function autoSnooze(ctx: Ctx) {
  const stale = await ctx.cap.mail.query({ filter: "awaiting_my_reply", olderThan: "48h" });
  for (const t of stale) await ctx.cap.mail.snooze({ threadId: t.id, until: "tomorrow_9am" });
}
```

It runs on the runtime's scheduler (or, later, on Manyshape-hosted edge workers for always-on behavior), holds exactly two capability handles, hits the same server-side authz as any UI click, and appears in the audit log. Cross-app pipelines ("invoice email → parse line items → `accounting.createEntry`") are the same mechanism with handles from two contracts - the user-level composability that sealed apps can never offer.

---

## 4. Registry, versioning, migration

### 4.1 Distribution
- Contracts are content-addressed, vendor-signed packages; the runtime verifies signatures against the vendor's registry identity (sigstore-style transparency log). Interface packs are user-signed.
- Semver with **contract-aware diffing**: the registry computes whether a release is additive (new caps/fields), compatible (renames with aliases), or breaking (removed/changed caps), and enforces honest version bumps mechanically.

### 4.2 The migration story (the make-or-break)
When `mail-app` goes 2.x → 3.0:
1. Runtime fetches the new contract + the registry's machine-readable diff.
2. Surfaces bind to capability IDs, so **additive changes require nothing** - surfaces keep running, and the agent may *offer* to use new primitives ("threads now have priority - want it on your cards?").
3. For breaking changes, the runtime **recompiles the surface from the intent spec** against the new contract. The intent spec survives by construction (it's contract-relative); the agent resolves what doesn't map (`snooze` signature changed → re-derive the auto-snooze workflow), re-runs the activation gate, and presents a diff for anything it had to reinterpret.
4. Rollback is trivial: previous surface + contract pin, one tap.

Vendors get an aggregate (opt-in, anonymized) view of migration health: "12% of intent specs reference the capability you're deprecating" - telemetry no vendor of sealed software has ever had about customization.

### 4.3 Cooperative by design
Manyshape only reshapes apps whose vendor ships or registers a capability contract. There is no scraping or synthetic-contract path for non-consenting third parties: the extension reshapes a page client-side, as the signed-in user, on data they can already see (the ad-blocker / userstyle footing), and only when a contract is present. This keeps the product on defensible legal ground and keeps vendors as allies. The adoption path is the registry and interface packs (§4.1, and README §3.4), not reverse-engineering.

---

## 5. Security model, summarized

| Threat | Defense |
|---|---|
| Malicious/hallucinated surface exfiltrates data | No ambient network; only declared capability handles; data-time policies (taint tracking) on sensitive fields |
| Surface tricks user into destructive action | `risk` annotations + activation-time policy requiring confirmation affordances + call-time gesture checks |
| Generated code escalates privileges | Authz is server-side in the authority plane; the client is untrusted by design |
| Prompt injection via app content (email body says "delete everything") | Agent operates only at customization time with user-visible diffs, never silently at data-read time; workflows are static once activated - content can't rewrite them |
| Malicious interface pack | Packs are intent specs, not code - applying one re-derives locally through the full activation gate; packs are signed and reputation-scored |
| Compromised vendor contract update | Signed releases + transparency log + staged rollout; breaking-change recompiles show diffs before adoption |

The through-line: **generated code is treated like a webpage, not like an installed app** - rich rendering, zero authority beyond explicit grants, and everything auditable.

