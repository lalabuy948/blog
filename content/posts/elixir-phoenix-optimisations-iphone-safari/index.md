---
title: "Elixir Phoenix Optimisations iPhone Safari"
date: 2026-02-14T16:07:30+01:00
draft: true
comments: true
cover: "/img/elixir-phoenix-optimisations-iphone-safari/image.png"
tags:
  - code
  - elixir
  - tutorial

seo:
  - phoenix liveview ios safari
  - liveview websocket reconnect
  - ios safari websocket freeze
  - phoenix visibilitychange fix
  - liveview longpoll fallback
  - phoenix static pages controller
  - mobile safari performance
  - phoenix 1.8.3 upgrade

seo_description: "Fix iOS Safari freezing and slow reconnects in Phoenix LiveView. Covers upgrading to Phoenix 1.8.3 for visibilitychange support, tuning longPollFallbackMs, reducing CSS blur performance issues, and converting static pages from LiveView to controllers for faster mobile experience."
---

If you're running a Phoenix LiveView application and your iOS Safari users are experiencing frozen pages, slow reconnects, or 10+ second hangs after returning from sleep — this article is for you.

I've been running a production Phoenix LiveView application. The app worked great on desktop, but iPhone users kept reporting the same thing: the page would freeze or hang for several seconds after switching back from another app or unlocking their phone. Sometimes the page would never recover at all.

Here's what was actually happening and how I fixed it.

## The Problem

The symptoms were consistent:

- User opens the app on iPhone Safari
- Switches to another app or locks the phone
- Returns to the page — it's frozen for 5-15 seconds
- Sometimes the page never reconnects and requires a manual refresh

This wasn't a bug in my code. It was a combination of how iOS Safari handles WebSockets, how Phoenix reconnects after sleep, and some surprising CSS performance issues.

## Root Cause Analysis

After digging through Phoenix issues, Elixir forum posts, and a lot of iOS Safari debugging, I found multiple layers to this problem.

### 1. Phoenix Reconnect Logic Before 1.8.2

When a phone sleeps, iOS Safari kills the WebSocket connection. Phoenix detects the error and immediately starts trying to reconnect — but the page is hidden. The browser throttles or stalls these background reconnection attempts. When the user returns, there's a stalled reconnect attempt blocking the main thread for seconds.

Phoenix 1.8.2 (PR [#6534](https://github.com/phoenixframework/phoenix/pull/6534)) fixed this by using the `visibilitychange` API. It stops reconnect attempts while the page is hidden and immediately reconnects when it becomes visible. This single fix is the biggest improvement.

### 2. Aggressive LongPoll Fallback

Phoenix LiveView has a `longPollFallbackMs` option that falls back to HTTP long-polling if the WebSocket handshake takes too long. The commonly recommended value of `2500ms` is too aggressive for mobile networks where the initial WebSocket handshake can be slow.

The problem compounds because the longpoll fallback flag is stored in `sessionStorage`. Once a client falls back, it stays on long-polling until the user opens a new tab.

### 3. Heavy CSS Blur Effects

This one surprised me. Large CSS `blur()` values on decorative background elements were causing mobile Safari to stall WebSocket connections for several seconds. The hero section had glow effects with `blur-[120px]` and `blur-[150px]` — purely decorative elements that were killing network performance on iOS.

### 4. Full-Page LiveView for Static Content

The home page was a single LiveView rendering everything — hero section, country grid, testimonials, FAQ, CTA. That meant a full WebSocket connection for a page where 90% of the content was static HTML. Every reconnect issue affected the entire page.

## The Fixes

### Upgrade Phoenix to 1.8.3

The most impactful change. Update `mix.exs`:

```elixir
{:phoenix, "~> 1.8.3"},
```

Then run:

```bash
mix deps.update phoenix
```

This also pulled in Bandit 1.10.2 and Plug 1.19.1 with their own fixes.

### Increase LongPoll Fallback Timeout

In `app.js`, change the timeout from 2500ms to 5000ms:

```javascript
const liveSocket = new LiveSocket("/live", Socket, {
  longPollFallbackMs: 5000,
  params: {_csrf_token: csrfToken},
  hooks: hooks,
})
```

If all your users are on modern browsers, you can also remove `longPollFallbackMs` entirely to disable the fallback.

### Add Visibility Change Safety Net

Even with Phoenix 1.8.3's built-in fix, I added an extra safety net in `app.js` that force-reconnects when the page becomes visible:

```javascript
document.addEventListener("visibilitychange", () => {
  if (document.visibilityState === "visible") {
    if (liveSocket.socket.connectionState() !== "open") {
      liveSocket.socket.disconnect(() => liveSocket.socket.connect())
    }
  }
})
```

This covers edge cases where the built-in reconnect logic might not fire quickly enough.

### Reduce CSS Blur on Mobile

Replace extreme blur values with responsive alternatives. Mobile gets a lighter blur, desktop keeps the full effect:

```html
<!-- Before -->
<div class="bg-primary/20 blur-[120px] rounded-full" />

<!-- After -->
<div class="bg-primary/20 blur-3xl sm:blur-[120px] rounded-full" />
```

`blur-3xl` is 72px — still looks great on mobile but doesn't stall the browser's network stack.

### Convert Static Pages to Controllers

This was the architectural fix. Instead of rendering the entire home page as a LiveView (with a WebSocket connection), I converted it to a standard Phoenix controller. Only the interactive countries filter section uses a LiveView, embedded via `live_render`:

```elixir
# Router - before
live "/", HomeLive, :index

# Router - after
get "/", PageController, :home
```

The controller template renders all static sections as plain HTML and embeds just the interactive part:

```heex
<Layouts.app flash={@flash} current_scope={@current_scope} transparent_header>
  <%!-- Static hero, testimonials, FAQ, CTA sections --%>
  ...

  <%!-- Only this section needs a WebSocket --%>
  {live_render(@conn, EasyIncWeb.CountriesFilterLive, id: "countries-filter")}

  ...
</Layouts.app>
```

The result: 90% of the home page loads instantly as static HTML. No WebSocket means no reconnect problems for that content. Only the countries filter maintains a lightweight LiveView connection.

For JS hooks that were previously managed by LiveView (like a 3D globe animation and scroll-aware header), I added standalone initializers that run on `DOMContentLoaded`:

```javascript
document.addEventListener('DOMContentLoaded', () => {
  initGlobeStandalone()
  initScrollHeaderStandalone()
})
```

These work alongside the LiveView hooks — the standalone init handles controller pages, the hooks handle LiveView pages.

## Other Things Worth Knowing

### Bandit vs Cowboy

There are reports of iOS WebSocket issues being resolved by switching from Bandit to Cowboy. If you're still having problems after the above fixes, it's worth testing:

```elixir
# In mix.exs, replace {:bandit, "~> 1.5"} with:
{:plug_cowboy, "~> 2.7"}

# In config.exs:
config :my_app, MyAppWeb.Endpoint,
  adapter: Phoenix.Endpoint.Cowboy2Adapter
```

### iCloud Private Relay

Safari has a bug where iCloud Private Relay sends CONNECT requests instead of proper WebSocket upgrade requests. This breaks WebSocket connections on some infrastructure (Cloudflare, GCP load balancers, nginx). There's no server-side fix — it's an Apple bug that appears to be resolved in Safari 26.0.1.

### TLS Configuration

Safari's WebSocket implementation is strict with TLS certificate validation and has documented issues when exclusively using TLS 1.3. Make sure your server supports both TLS 1.2 and 1.3.

## Summary

The priority order for fixing iOS Safari LiveView performance:

1. **Upgrade Phoenix to 1.8.3+** — gets you the `visibilitychange` reconnect fix
2. **Convert static pages to controllers** — eliminates unnecessary WebSocket connections
3. **Tune `longPollFallbackMs`** — increase to 5000ms or disable entirely
4. **Audit heavy CSS** — large `blur()` and `backdrop-filter` values stall mobile networking
5. **Add `visibilitychange` safety net** — belt and suspenders for reconnection
6. **Test Bandit vs Cowboy** — if issues persist after everything else

The fundamental lesson: not every page needs a WebSocket. Phoenix makes it easy to use LiveView for everything, but the right architecture is static HTML for static content and LiveView only where you need interactivity. Your iOS users will thank you.
