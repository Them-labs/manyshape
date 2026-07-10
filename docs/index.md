Manyshape is an open (MIT) delivery stack where applications ship as **capability contracts** - schemas, typed actions, policies, and a reference UI - and each user's agent compiles a personal interface ("surface") on top. The vendor's backend stays closed and authoritative; the interface becomes a regenerable, user-owned artifact.

## The pieces

| Piece | What it is |
|---|---|
| [Capability contract](./contracts.md) | What a vendor ships: schemas, capabilities, policies, design tokens, docs |
| [Surface](./surfaces.md) | A generated interface (vanilla JS or React) running in a locked sandbox |
| [Vendor SDK](./vendor-sdk.md) | `@manyshape/sdk` - validate contracts, mount an authority plane on Express |
| [Chat SDK](./chat-sdk.md) | `@manyshape/chat-sdk` - one script tag gives your app the ◈ customize bubble |
| [Platform](./platform-api.md) | Accounts, sessions, the contract registry, cloud experiences, hosted agent |
| Runtime | The browser host: capability bridge, activation gate, sandbox (see [Security](./security.md)) |
| Extension | Chrome MV3 doorway - remix any page that ships or registers a contract |

## The loop, in one paragraph

A user tells the agent what they want. The agent reads the app's contract and reference surface, and compiles a new surface honoring the user's **entire intent history**. The activation gate then checks it - declared capabilities ⊆ contract, a static scan for ambient I/O, and a boot test in a hidden sandbox - before it replaces the visible interface. At runtime the surface can only act through the capability bridge, and the vendor's authority plane re-validates every call server-side. Signed-in users get the surface + intent spec saved to their account as an **experience**, which follows them across the runtime, embedded chat SDKs, and the browser extension.

## Who it's for

New here for the why rather than the how? See [Use cases & customers](./use-cases.md): the customer segments, what they build, and the beachhead.

Start with [Getting started](./getting-started.md).
