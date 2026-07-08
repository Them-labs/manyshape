A surface is a single self-contained HTML fragment - usually compiled by the agent, but you can write one by hand. It runs in a sandboxed iframe with **no network, no storage, and no parent DOM access**; its only I/O is the injected `facet` API.

## File format

```html
<!--surface
name: focus-list
caps: mail.query, mail.archive
framework: react
intent: One-line restatement of the cumulative intent.
-->
<style>…</style>
<div id="root"></div>
<script type="text/jsx">…</script>
```

The header comment is load-bearing: `caps` declares every capability the surface may call (the bridge refuses anything else), and `framework` is `vanilla` (default) or `react`.

## The facet API (vanilla)

```js
const threads = await facet.cap("mail.query", { filter: "inbox" });
facet.timeAgo(iso)   // "3h" / "2d"
facet.user           // current app-user id
```

`facet.cap` resolves with the capability's output or rejects with the policy/authz error. Render data with `textContent` - never interpolate into `innerHTML`.

## React surfaces

Declare `framework: react` and write JSX in `<script type="text/jsx">` - it is transpiled with esbuild **before** the gate runs. The runtime injects `window.React` / `window.ReactDOM` (the React API via a ~25KB Preact-compat bundle) - never import or load a framework yourself; the sandbox has no network to fetch one.

```jsx
function App() {
  const [threads, refresh] = useCap("mail.query", { filter: "inbox" });
  if (!threads) return <p>Loading…</p>;
  return threads.map((t) => (
    <div key={t.id}>{t.subject}
      <button onClick={async () => { await facet.cap("mail.archive", { threadId: t.id }); refresh(); }}>
        Done
      </button>
    </div>
  ));
}
ReactDOM.render(<App />, document.getElementById("root"));
```

`useCap(capId, args)` → `[data, refresh]` is provided for reads; call `refresh()` after mutations. `dangerouslySetInnerHTML` is forbidden.

## The activation gate

Every surface passes four checks before it replaces anything visible:

1. **Header parse** - valid header with a non-empty `caps` list.
2. **Caps ⊆ contract** - every declared capability exists in the contract.
3. **Static policy scan** - rejects `fetch`, `XMLHttpRequest`, `WebSocket`, `EventSource`, `sendBeacon`, dynamic `import()`, and external `src`/`href` URLs. (Defense-in-depth: the sandbox CSP is the real wall.)
4. **Conformance boot** - the surface is loaded in a hidden staging iframe and must make its first successful capability call within 8 seconds with zero uncaught JS errors.

Failure at any step keeps the previous surface live - a user is never left on a broken screen. A newer activation cleanly cancels an in-flight one.

## Rules the agent is held to

Render immediately and query on load; style only with the contract's design tokens (`var(--page)`, `var(--ink)`, …); honor the **entire** intent history, not just the last utterance; remix the vendor's reference surface rather than generating from a blank page.
