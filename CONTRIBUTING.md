# Contributing to Manyshape

Thanks for your interest. The project is early; the highest-leverage contributions right now:

- **Contracts for real apps** - write a capability contract + authority-plane adapter for software you use (see [packages/sdk](https://github.com/Them-labs/manyshape-sdk) and the [mail example](https://github.com/Them-labs/manyshape-examples)).
- **Runtime hardening** - the [known v0 shortcuts](https://github.com/Them-labs/manyshape-examples) are an honest to-do list (real auth, trusted gesture attestation, rate limiting).
- **Surface frameworks** - vanilla and React are supported today; the pattern in [packages/surface-sdk](https://github.com/Them-labs/manyshape-surface-sdk) extends to others (Svelte, Vue) if you can keep the injected runtime small.
- **Conformance tooling** - executable acceptance criteria that gate generated surfaces.

## Setup

```sh
npm install          # workspaces: packages/* + mvp
npm run mvp          # reference app on http://localhost:8484
npm run dev          # landing site on http://localhost:8321
```

The agent needs a key in `mvp/.env` (see [mvp/.env.example](https://github.com/Them-labs/manyshape-examples)); without one the MVP runs in canned-fallback mode, which is fine for runtime/SDK work.

## Ground rules

- Security model changes (sandbox, CSP, bridge, gate, authority router) need a written threat rationale in the PR description.
- Surfaces are untrusted by construction. Never add a capability for generated code that bypasses the bridge or the server-side checks.
- Keep the in-sandbox payloads small - the React runtime is ~25KB minified and should stay in that neighborhood.

Manyshape is stewarded by Them Labs, Inc. By contributing you agree your contributions are licensed under the [MIT License](LICENSE).
