# Live-delivery options on React Router 8

Research for [#7](https://github.com/knn90/cohort-005-project/issues/7). Facts only; [#8](https://github.com/knn90/cohort-005-project/issues/8) decides.

Question: what can React Router 8 (framework mode) plus `@react-router/serve` (one Node process, SQLite through Drizzle) use to push new data to an open page within a few seconds? The options are Server-Sent Events (SSE) from a resource route, WebSocket, or polling.

## The stack as it stands

- `react-router`, `@react-router/node`, `@react-router/serve` and `@react-router/dev` are all 8.3.0 (`package.json`). Production runs `react-router-serve ./build/server/index.js`. Dev runs `react-router dev`, which is Vite. Node is v22.14.0 locally.
- `react-router-serve` (`node_modules/@react-router/serve/dist/cli.js`) is a fixed Express 5 app with `compression()` (line 113), `morgan`, static files, and `app.all("/{*splat}", createRequestHandler(...))`. It calls `app.listen(port)`: plain HTTP/1.1, no `upgrade` handling, and no way to reach the `http.Server`.
- Routes are listed explicitly in `app/routes.ts`. There are already resource routes under `api/*` (for example `api.video-tracking.ts`), with no default export.
- `app/root.tsx` holds only layout and error UI, with no loader. The per-user data loads in `routes/layout.app.tsx` `loader`.
- Session: `app/lib/session.ts` uses `createCookieSessionStorage` (cookie `cadence_session`, `httpOnly`, `sameSite: "lax"`, signed). `getCurrentUserId(request)` reads the `Cookie` header of a Fetch `Request`. `app/lib/access.server.ts` builds on it. Auth depends only on that request header.
- DB: one `better-sqlite3` connection in WAL mode (`app/db/index.ts`). It is synchronous and single-process by design (ADR 0006).

## 1. SSE from a resource route

**How it plugs in.** Add a route with a `loader` and no default export, for example `route("api/notifications/stream", ...)`. The loader returns `new Response(readableStream, { headers: { "Content-Type": "text/event-stream", ... } })`. Resource routes may return a raw `Response` ([RR resource routes](https://reactrouter.com/how-to/resource-routes)). The client opens `new EventSource("/api/notifications/stream")` in a `useEffect` and calls `revalidate()` or updates state when a message arrives. No new dependencies are needed, and the same code runs under `react-router dev`.

- The Express adapter already handles SSE. In `@react-router/express/dist/index.js`, `sendRemixResponse` calls `res.flushHeaders()` when the Content-Type matches `text/event-stream`. `writeReadableStreamToWritable` (`@react-router/node`) calls `writable.flush()` after every chunk, so the `compression()` that serve adds does not hold events back.
- `compression` skips any response with `Cache-Control: no-transform` (`compression/index.js` `shouldTransform`). Sending `Cache-Control: no-cache, no-transform` takes compression out of the path completely.
- Cleanup: the adapter wires `res.on("close") → controller.abort()` into `request.signal`. The loader should listen for `request.signal`'s `abort` event to unsubscribe and close the stream.
- Server-side source of events: one process (ADR 0006) allows an in-memory `EventEmitter` fan-out, called wherever a Notification row is written. The alternative is a short server-side SQLite poll per open stream. Neither works across processes.

**Session/auth.** `EventSource` is a GET to the same origin, so the browser sends the `cadence_session` cookie (credentials mode is same-origin; `withCredentials` only matters cross-origin, [WHATWG SSE](https://html.spec.whatwg.org/multipage/server-sent-events.html)). The loader calls `getCurrentUserId(request)` exactly as other loaders do, and returns 401 if there is no user. The session is read **once, at connect**. A later logout or DevUI user switch (`api/switch-user`) does not affect an open stream, which keeps the old user until it reconnects. The client has to close and reopen the stream when the user changes.

**Proxy/buffering/timeout gotchas.**
- **nginx** buffers proxied responses by default (`proxy_buffering on`). The upstream can turn this off per response with the header `X-Accel-Buffering: no`. `proxy_read_timeout` defaults to 60s and is an idle timeout between reads ([nginx proxy module](https://nginx.org/en/docs/http/ngx_http_proxy_module.html)).
- The spec advises a comment line (`:`) "every 15 seconds or so" because legacy proxies drop idle connections ([WHATWG SSE](https://html.spec.whatwg.org/multipage/server-sent-events.html)). A heartbeat also covers nginx's 60s default.
- **Node**: `server.timeout` defaults to 0 (off). `requestTimeout` (300s) only covers *receiving* the request, which for a body-less GET ends at once. `keepAliveTimeout` only applies after a response finishes ([Node http](https://nodejs.org/docs/latest-v22.x/api/http.html)). None of these cuts off a long-lived response by default.
- **HTTP/1.1 connection cap**: without HTTP/2, browsers allow about 6 connections per browser and domain *across all tabs*. Chrome and Firefox mark this "Won't fix" ([MDN EventSource](https://developer.mozilla.org/en-US/docs/Web/API/EventSource)). `react-router-serve` only speaks HTTP/1.1, so with one stream per tab, about 6 tabs use up the cap and block every other request to the origin. Workarounds: HTTP/2 at a proxy in front, or one shared stream through a SharedWorker or BroadcastChannel leader tab (the spec suggests a shared worker).

**Reconnect.** This is built into the browser. After a network error the browser waits the reconnection time ("a few seconds", which the server can change with `retry:`) and reconnects, sending the last `id:` as a `Last-Event-ID` header, so the server can replay missed events. A non-200 status or a wrong Content-Type is a *fatal* failure with no reconnect, and 204 is the server's way to stop reconnects ([WHATWG SSE](https://html.spec.whatwg.org/multipage/server-sent-events.html)). A 401 from an expired session therefore ends the stream for good, which is the right outcome.

**Cost/complexity.** No new dependencies. About 1 resource route, 1 client hook and 1 in-process emitter. Each open tab holds one socket and a little memory. Latency is effectively immediate. The work is in the heartbeat, abort cleanup, the tab cap, and replay from `Last-Event-ID` or from an "unread since" query.

## 2. WebSocket

**How it plugs in.** It does not fit the current stack:
- `react-router-serve` exposes no `http.Server` and registers no `upgrade` listener. In Node, upgrade requests are ignored unless something listens for `'upgrade'` ([Node http](https://nodejs.org/docs/latest-v22.x/api/http.html)).
- Node core ships only a *client* `WebSocket` global (unflagged v22.0.0, stable v22.4.0, [Node globals](https://nodejs.org/docs/latest-v22.x/api/globals.html)). There is no WebSocket server in core, so the server side needs a library such as `ws`, which is not installed.
- So it means **replacing `react-router-serve` with a custom server**. The official `node-custom-server` template (remix-run/react-router-templates) is Express with `compression`, `morgan`, and `createRequestHandler` in `server/app.ts`. In dev it runs Vite in `middlewareMode` and `ssrLoadModule`s the app. You would attach `ws` to the `http.Server` that `app.listen` returns. That changes the `dev`/`start` scripts, and the socket lives outside React Router's route and loader model, so there are no route modules or typegen for it.

**Session/auth.** Same-origin handshakes carry cookies, but the handshake is a raw Node `IncomingMessage`, not a Fetch `Request`. You would call `sessionStorage.getSession(req.headers.cookie)` yourself. That function is currently private to `session.ts`, so it would need exporting or wrapping. As with SSE, auth is checked once at the handshake. The server should also check `Origin` against cross-site WebSocket hijacking (RFC 6455 §10.2). `SameSite=Lax` already keeps the cookie off cross-site subresource requests.

**Proxy/buffering/timeout gotchas.** nginx must pass `Upgrade` and `Connection` explicitly (they are hop-by-hop), and it closes connections idle for 60s unless `proxy_read_timeout` is raised or the server sends ping frames ([nginx WebSocket](https://nginx.org/en/docs/http/websocket.html)). Buffering and compression do not apply once upgraded.

**Reconnect.** None built in. The browser `WebSocket` stays closed after `close`/`error`. Reconnect, backoff, heartbeat and catch-up are all application code.

**Cost/complexity.** The highest of the three: a new dependency, a custom server for dev and prod, a hand-written reconnect and heartbeat, and auth outside the route model. Its one advantage, two-way messaging, is not needed: delivery is server→client only, and client→server already goes through actions and fetchers.

## 3. Polling (`useRevalidator` or a fetcher on an interval)

**How it plugs in.** Pure client code with no server changes. Two shapes:
- `useRevalidator().revalidate()` in a `setInterval`. The docs name "polling on an interval" as a use case but ship no built-in interval or focus behaviour ([useRevalidator](https://reactrouter.com/api/hooks/useRevalidator)). It re-runs **every active route loader**: `revalidate()` → `interruptActiveLoads()` sets `isRevalidationRequired = true`, so `getMatchesToLoad` gives each match `defaultShouldRevalidate = true` (`react-router/dist/development/lib/router/router.js` around lines 473, 1091, 1963). Routes can opt out with `shouldRevalidate`, which affects only that route ([route module](https://reactrouter.com/start/framework/route-module)). Here that means `layout.app.tsx` plus the page loaders on every tick.
- `useFetcher().load("/api/notifications")` in a `setInterval`, against a small resource route. This runs only that loader and keeps the page loaders untouched. It is the narrower option.
- Use the Page Visibility API (`document.hidden`/`visibilitychange`) to pause polling in background tabs and fetch at once when the tab becomes visible again.

**Session/auth.** Each poll is an ordinary same-origin request with the cookie, read through `getCurrentUserId` on every tick. User switches and logout take effect on the next poll with no extra code.

**Proxy/buffering/timeout gotchas.** None. These are short ordinary requests: no buffering, idle-timeout or connection-cap issues.

**Reconnect.** Not needed: each poll stands alone. A failed poll just retries on the next tick. Missed items are caught up naturally if the endpoint returns current state, such as unread Notifications, rather than deltas.

**Cost/complexity.** The lowest complexity, about 1 hook and optionally 1 resource route. Latency is up to one interval: about 3–5s fits the map's "within a few seconds; polling is acceptable". Load is (open tabs ÷ interval) requests, each a synchronous indexed SQLite read, mostly returning "nothing new". `revalidate()` multiplies that by the number of loaders on the page, and `fetcher.load` does not.

## Comparison

| | SSE resource route | WebSocket | Polling (fetcher) |
|---|---|---|---|
| Works with `react-router-serve` as is | Yes (adapter flushes `text/event-stream`) | No, needs a custom server + `ws` | Yes |
| New deps | None | `ws` + custom server | None |
| Latency | ~Immediate | ~Immediate | ≤ interval |
| Session reaches it | Cookie on GET, `getCurrentUserId(request)`, once at connect | Cookie on handshake, parse manually, once at connect, check Origin | Cookie on every request |
| User switch / logout | Stale until reconnect | Stale until reconnect | Next tick |
| Proxy gotchas | nginx buffering (`X-Accel-Buffering: no`), 60s idle → heartbeat | Upgrade headers, 60s idle → ping | None |
| Browser limits | ~6 per domain across tabs on HTTP/1.1 | Not the SSE/HTTP cap (not researched further) | None |
| Reconnect | Built in, `Last-Event-ID` | Hand-written | N/A |
| Multi-process later | Needs a cross-process pub/sub | Needs a cross-process pub/sub | Works unchanged (reads DB) |
| Complexity | Medium | High | Low |

Not verified by running code. Every claim above comes from the cited docs or the installed 8.3.0 package source.
