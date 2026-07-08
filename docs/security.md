The core bet: **security never depends on the generated code behaving.** Generated interfaces are treated like webpages, not installed apps - rich rendering, zero authority beyond explicit grants.

## The layers

1. **Sandbox** - surfaces run in an `srcdoc` iframe with `sandbox="allow-scripts"` and no `allow-same-origin`: opaque origin, no cookies, no storage, no parent DOM.
2. **CSP** - `default-src 'none'` injected into every surface. Even code that slips past the static scan cannot reach the network.
3. **Declared capabilities** - the bridge only forwards calls the surface declared *and* the contract contains. A surface's blast radius is capped at its declared set.
4. **Authority plane** - the vendor's server re-validates everything per call: the capability exists, the user is authorized, inputs are valid, and call-time policies hold (`gesture: true` actions reject without a fresh user gesture).
5. **Activation gate** - fails closed. Header parse → caps ⊆ contract → static ambient-I/O scan → sandboxed boot test. A failing surface never activates; the previous one stays live.

Prompt injection has a bounded story too: app *content* (e.g. email bodies) never enters the generation prompt - only the user's intents do - and anything injected into a rendered surface is still inside the sandbox, still limited to declared caps.

## Threats and answers

| Threat | Defense |
|---|---|
| Malicious/hallucinated surface exfiltrates data | No ambient network (CSP + sandbox); only declared capability handles |
| Surface calls something it shouldn't | Bridge refuses undeclared caps; authority plane re-authorizes server-side |
| Generated code escalates privileges | The client is untrusted by design; authz lives server-side |
| Surface tricks a destructive action | `risk` annotations + gesture policy on `external` capabilities |
| Broken surface strands the user | Gate fails closed; one-tap reset to the reference surface |

## Known demo-grade gaps

Be honest about what the reference implementation does **not** yet harden:

- **App-user identity is a self-asserted header** (`x-facet-user`) in the demo vendor app. Real vendors must resolve identity from their own sessions in `getUser`.
- **Gesture age is self-reported** by the (untrusted) guest SDK - a hostile surface could spoof it. Real enforcement needs runtime-attested user activation.
- **Platform auth has no email verification** - an unregistered email can be claimed by whoever signs up first. Magic-link verification is the next step.
- **The static scan is regex-based** and bypassable on its own - it exists for fast, explainable rejects; CSP + sandbox are the actual wall.
- No rate limiting on the reference platform endpoints.

These are listed in the repo as contribution-welcome items; none of them weaken the core sandbox/authority separation.
