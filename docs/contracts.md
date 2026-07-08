A contract is what your app ships so agents can build interfaces for it: what your data means, what actions exist, what invariants must hold, and what your design language is. It is JSON, semver'd, and validated by `validateContract` from [`@manyshape/sdk`](./vendor-sdk.md).

```jsonc
{
  "id": "mail-app",                // namespaced app id
  "name": "Mail",
  "version": "1.0.0",              // semver - drives migration behavior
  "docs": "Agent-readable description of what the app means…",
  "schemas": { "thread": { /* JSON Schema */ } },
  "capabilities": [ /* see below */ ],
  "policies": [ /* declared invariants */ ],
  "conformance": [ { "id": "boots", "docs": "…first cap call within 8s…" } ],
  "tokens": { "page": "#f8f7f9", "ink": "#211b24", "radius": "12px", /* … */ }
}
```

## Capabilities

Capabilities are the **only** way any surface touches your app. Each is a typed action with declared semantics:

```jsonc
{
  "id": "mail.snooze",                    // namespaced: app.verb
  "input": { "threadId": "string", "until": "ISO date-time" },
  "output": "{ ok: boolean }",
  "risk": "reversible",                   // read | reversible | destructive | external
  "gesture": false,                       // true → requires a fresh user gesture
  "docs": "Hide a thread until the given time."   // agents read this - required
}
```

- **`risk`** drives policy generically - e.g. a confirmation policy can bind to every `destructive` capability without naming them.
- **`gesture: true`** activates the call-time policy: the authority plane rejects the call unless it arrives within 1500ms of a real user gesture. Use it for anything external (sending, paying, posting).
- **`docs` are required.** Agents decide when and how to use a capability from this text - write it like you'd write a good tool description.

## Policies

Declared invariants, each with an `enforcement` binding time:

| Enforcement | Example | Enforced by |
|---|---|---|
| `activation-time` | no ambient network in surfaces | gate: static scan + sandbox CSP |
| `call-time` | external actions require a gesture | authority router |
| `activation-time + call-time` | declared-caps-only | bridge + gate |

## Design tokens

Every key in `tokens` is injected into the sandbox as a CSS custom property (`--page`, `--ink`, `--radius`, …). Generated surfaces are instructed to style exclusively with them, so every shape of your app still looks like your app.

## Distribution

Serve the contract at a URL (the reference app uses `GET /api/contract`, returning `{ contract, referenceSurface }`), declare it on your pages for the extension:

```html
<link rel="manyshape-contract" href="/api/contract"
      data-cap-base="/api/cap" data-guest-sdk="/guest-sdk.js"
      data-react-runtime="/react-runtime.js">
```

and publish it to the [registry](/docs/platform-api/#registry) so clients can resolve it by origin.
