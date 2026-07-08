The chat SDK is the embedded doorway: a floating bubble inside your own app where users sign in to their Manyshape account, describe the interface they want, and your app is rebuilt in place - sandboxed, gated, and saved to their account.

## Embed

```html
<div id="app-root"><!-- your normal interface --></div>

<script src="/chat-sdk.js"></script>
<script>
  Manyshape.mount({
    target: "#app-root",                    // container the surface replaces
    contractUrl: "/api/contract",
    capBase: "/api/cap",
    appUser: currentUser.id,                // your app's user identity
    platform: "https://platform.manyshape.com",
  });
</script>
```

Serve `chat-sdk.js` from the `@manyshape/chat-sdk` package (the reference app exposes it at `/chat-sdk.js`).

## Options

| Option | Default | Meaning |
|---|---|---|
| `target` | - (required) | CSS selector whose contents are replaced by activated surfaces |
| `contractUrl` | `/api/contract` | Returns `{ contract, referenceSurface }` |
| `capBase` | `/api/cap` | Authority-plane base path |
| `guestSdkUrl` | `/guest-sdk.js` | In-sandbox facet API |
| `reactRuntimeUrl` | `/react-runtime.js` | Injected for `framework: react` surfaces |
| `platform` | `http://localhost:8600` | Manyshape platform origin |
| `appUser` | `"jk"` | Your app's user id, sent as `x-facet-user` on capability calls |

## Behavior

- **Signed out**: the bubble opens to a sign-in form (accounts are created on the [platform](./platform-api.md)). Generation requires an account - it runs on the platform's hosted agent.
- **Signed in**: on mount, the user's most recent experience for this app **auto-applies**, so your app opens already shaped to them. The panel lists all their experiences for this app with one-click Apply.
- Every activation passes the same [gate](/docs/surfaces/#the-activation-gate) as the standalone runtime, mounts as a sandboxed iframe inside `target`, and is saved to the user's account.
- **Reset** restores your original DOM exactly as it was at mount time.

## Security notes

The surface iframe is `sandbox="allow-scripts"` with CSP `default-src 'none'` - it cannot touch your page, your cookies, or the network. Its capability calls go through the SDK's bridge to your `capBase` with the `appUser` you provided, and your authority plane still enforces authorization server-side. Treat `appUser` as advisory display identity, not proof - see [Security model](./security.md).
