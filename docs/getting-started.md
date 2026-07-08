## Prerequisites

- Node.js 20+
- MongoDB running locally (`brew services start mongodb-community`), or set `MONGODB_URI`
- Optional: a Gemini or Anthropic API key for live generation (without one, the demo serves canned surfaces)

## Run the stack

```sh
git clone https://github.com/Them-labs/manyshape-examples && cd manyshape-examples
npm install

cp mvp/.env.example mvp/.env    # paste GEMINI_API_KEY or ANTHROPIC_API_KEY
npm run mvp                     # demo mail app + runtime  → http://localhost:8484
node platform/server.js        # platform + dashboard     → http://localhost:8600
```

## Reshape your first app

1. Open **http://localhost:8484** - the standalone runtime hosting the demo mail app on its vendor reference surface.
2. Type an intent: *"I live in my inbox. Make it a task board where archiving a thread completes it."* and hit **Rebuild my interface**.
3. Watch the activation gate run its four checks, then the new surface goes live. Ask for React explicitly ("build it as a React surface with tabs…") to get a `framework: react` surface.
4. Switch the user (jk → maya) - each user's intent spec and surface are independent. Same app, same contract, different products.

## Add an account

Create an account at **http://localhost:8600** (email + password), then sign in from the runtime header. From then on generation runs through the platform's hosted agent and every activated surface is saved as a cloud **experience** - visible in the dashboard and applicable from the vendor page's chat bubble (http://localhost:8484/vendor.html) and the [browser extension](https://github.com/Them-labs/manyshape-extension).

## Where to go next

- Ship your own app: [Capability contracts](./contracts.md) + [Vendor SDK](./vendor-sdk.md)
- Embed customization in an existing app: [Chat SDK](./chat-sdk.md)
- Understand what generated code can and cannot do: [Security model](./security.md)
