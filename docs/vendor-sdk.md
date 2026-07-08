```sh
npm install @manyshape/sdk express
```

The vendor SDK turns an existing backend into a Manyshape **authority plane**: your handlers run behind the contract's call-time policies, and all authorization stays server-side, below any interface.

## `validateContract(contract)`

Returns an array of error strings (empty = valid). Checks semver, namespaced capability ids, `risk`/`gesture` annotations, required capability docs, design tokens, and policy shape. Run it at boot and in CI.

## `createAuthorityRouter({ contract, getUser, handlers })`

An Express router that mounts `POST /:capId` for every capability:

```js
import express from "express";
import { validateContract, createAuthorityRouter, CapError } from "@manyshape/sdk";
import contract from "./my-app.contract.json" with { type: "json" };

const errors = validateContract(contract);
if (errors.length) throw new Error(errors.join("\n"));

const app = express();
app.use("/api/cap", createAuthorityRouter({
  contract,
  getUser: (req) => resolveSession(req),      // your real auth - null rejects with 401
  handlers: {
    "todo.list":    async ({ user }) => db.list(user),
    "todo.archive": async ({ user }, { id }) => {
      if (!(await db.owns(user, id))) throw new CapError(404, "no such item");
      return db.archive(user, id);
    },
  },
}));
```

What the router does before your handler runs:

1. **404** for capabilities not in the contract.
2. **401** when `getUser` returns null.
3. **403** for `gesture: true` capabilities called without a fresh user gesture (`x-facet-gesture-age` within 1500ms).
4. Parses `{ args }` from the JSON body and calls `handlers[capId]({ user, cap, req }, args)`.

Throw `new CapError(status, message)` for typed failures; anything else becomes a 500. Handler return values are wrapped as `{ ok: true, data }`.

## Gate primitives

`parseSurfaceHeader(source)` and `staticPolicyScan(source)` are exported for server-side pre-checks and CI - reject a surface with ambient I/O before it ever reaches a client. The [agent package](https://github.com/Them-labs/manyshape-agent) uses both after every generation.

## Serving the rest

Besides `/api/cap`, a vendor app serves its contract + reference surface (`GET /api/contract` → `{ contract, referenceSurface }`) and the in-sandbox assets from `@manyshape/surface-sdk` (`/guest-sdk.js`, and `/react-runtime.js` bundled from `react-entry.js`). See [mvp/server.js](https://github.com/Them-labs/manyshape-examples/blob/main/server.js) for the complete reference wiring, including publishing to the registry at boot.
