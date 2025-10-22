---
title: "Elixir Liveview Socket Reconnect"
date: 2025-10-22T18:22:33+02:00
draft: false
comments: true
cover: "/img/elixir-liveview-socket-reconnect/phase_VH.jpg"
tags:
    - elixir
    - code
    - work
seo:
    - phoenix liveview reconnection
    - liveview socket disconnect
    - phoenix websocket stale data
    - liveview mount counter
    - phoenix iot bug fix
seo_description: "Deep dive into fixing a critical Phoenix LiveView bug where stale form data was sent after WebSocket reconnection, causing hardware resets. Learn how to use the _mounts counter to prevent queued events from corrupting IoT device state."
---

## The Phoenix LiveView Reconnection Bug: A Deep Dive

I encountered a critical bug in a Phoenix LiveView settings page that controlled hardware via MQTT. When the WebSocket connection dropped and LiveView automatically reconnected, stale form values were being sent back to the hardware, overwriting the current hardware state.

<!--more-->

**What Was Happening:**

1. User has Phase Horizontal = 22 on hardware
2. Browser shows Phase Horizontal = 22
3. WiFi disconnects briefly
4. While disconnected, user changes hardware to Phase Horizontal = -25
5. Browser reconnects (LiveView remounts)
6. Browser still thinks value is 22 (stale)
7. Form sends "update" event with stale value 22
8. Hardware gets reset to 22 ❌

**MQTT Logs Showing the Problem:**

```
cy-vp4/videoproc/status/volatile/genlock_phase -25 10  # Hardware's current value
cy-vp4/videoproc/action/set/genlock genlock 22 10      # Stale value sent! ❌
```

This caused the hardware (an RCP - Remote Control Panel) to reset to old values whenever the network hiccupped.
### Understanding LiveView's Lifecycle

When a LiveView loses connection:

1. WebSocket disconnects (network issue, sleep, etc.)
2. Client queues any events that happen while disconnected
3. Client automatically reconnects
4. Server calls `mount/3` again (fresh start)
5. Client replays queued events ⚠️

**The problem:** The queued events contain stale form values from before the disconnect!

**Visual Example:**

- **T0:** Form shows: `phase_horizontal = 22`
- **T1:** WiFi disconnects
- **T2:** Hardware changes to: `phase_horizontal = -25`
- **T3:** User types "25" in form (queued, not sent)
- **T4:** WiFi reconnects → `mount/3` runs again
- **T5:** Queued event fires: `phx-change` with value = 22 (STALE!)
- **T6:** Server sends 22 to MQTT → Hardware reset ❌

### Attempted Solutions (And Why They Failed)

#### Attempt 1: The `recently_reconnected` Flag

```elixir
def mount(_params, _session, socket) do
  {:ok, assign(socket, :recently_reconnected, false)}
end

def handle_event("update_video", params, socket) do
  unless socket.assigns.recently_reconnected do
    # Send to MQTT
  end
  {:noreply, socket}
end
```

**Why it failed:** I set `recently_reconnected` to `false` on mount, so it was never `true` during reconnection! I had no way to detect when a reconnection actually happened.

#### Attempt 2: JavaScript Hook Detection

```javascript
const ReconnectionHook = {
  reconnected() {
    this.pushEvent("reconnected", {});
  }
}
```

```elixir
def handle_event("reconnected", _params, socket) do
  Process.send_after(self(), :clear_flag, 2000)
  {:noreply, assign(socket, :recently_reconnected, true)}
end
```

**Why it failed:**

- Race condition: JavaScript hook might fire after queued form events already processed
- Unreliable: Doesn't catch all reconnection scenarios
- Client-side dependency: If hook fails to load or execute, protection is gone
- Timing issues: No guarantee of execution order

**Example race condition:**

```
T0: Reconnect
T1: mount/3 runs → recently_reconnected = false
T2: Queued form event → processes normally → sends to MQTT ❌
T3: JS hook fires → sets recently_reconnected = true (too late!)
```

#### Attempt 3: Always Block on Mount

```elixir
def mount(_params, _session, socket) do
  if connected?(socket) do
    Process.send_after(self(), :clear_flag, 3000)
  end
  {:ok, assign(socket, :recently_reconnected, connected?(socket))}
end
```

**Why it didn't work well:**

- Blocked updates even on the first connection
- Users had to wait 3 seconds before the form worked
- Poor initial user experience
- No distinction between first visit and reconnection

### The Robust Solution: Using `_mounts` Counter

Phoenix LiveView provides a built-in reconnection counter called `_mounts` in the connection parameters.

#### How It Works:

```elixir
def mount(_params, _session, socket) do
  # Get LiveView's official reconnection counter
  is_reconnect = connected?(socket) &&
                 (get_connect_params(socket)["_mounts"] || 0) > 0

  # Only block events if this is a reconnection
  if is_reconnect do
    Process.send_after(self(), :clear_reconnect_flag, 500)
  end

  {:ok, assign(socket, :ignoring_reconnect_events, is_reconnect)}
end
```

**Key insight:** `_mounts` starts at 0 on first connection, then increments on each reconnection:

| Connection Type | `_mounts` Value | `is_reconnect` | Behavior |
|----------------|----------------|----------------|----------|
| First visit | 0 | `false` | Events work immediately ✅ |
| 2nd connection | 1 | `true` | Events blocked for 500ms ✅ |
| 3rd connection | 2 | `true` | Events blocked for 500ms ✅ |

#### Guard Clause to Block ALL Events

The critical piece: pattern match to block all events during reconnection:

```elixir
# This MUST come BEFORE other handle_event clauses
def handle_event(_event, _params, %{assigns: %{ignoring_reconnect_events: true}} = socket) do
  {:noreply, socket}
end

# Now your normal handlers
def handle_event("update_video", params, socket) do
  # This only runs if ignoring_reconnect_events is false
  {:noreply, update_video(socket, params)}
end

def handle_event("update_network", params, socket) do
  # This also gets blocked during reconnection
  {:noreply, update_network(socket, params)}
end
```

**Why pattern matching works:** Elixir tries function clauses in order from top to bottom. If `ignoring_reconnect_events: true`, the first clause matches and returns immediately, never reaching the other handlers.

#### Example Flow During Reconnection:

```elixir
# T0: Reconnect happens
mount(_params, _session, socket)
  → _mounts = 1
  → is_reconnect = true
  → ignoring_reconnect_events = true
  → Schedule :clear_reconnect_flag in 500ms

# T1: Stale queued event arrives
handle_event("update_video", %{"phase" => "22"}, socket)
  → Pattern matches guard clause (ignoring_reconnect_events = true)
  → Returns immediately, does nothing ✅

# T2: Fresh MQTT data arrives
handle_info({:setting_updated, "genlock_phase", {-25, 10}}, socket)
  → Updates form with fresh value
  → Form now shows -25 ✅

# T3: 500ms elapsed
handle_info(:clear_reconnect_flag, socket)
  → ignoring_reconnect_events = false
  → Normal operation resumes ✅
```

#### Clear the Flag After Short Delay

```elixir
def handle_info(:clear_reconnect_flag, socket) do
  # After 500ms, resume normal event processing
  {:noreply, assign(socket, :ignoring_reconnect_events, false)}
end
```

**Why 500ms?** Long enough for:

- Stale queued events to arrive and get blocked
- Fresh MQTT data to arrive via `handle_info({:setting_updated, ...})`
- Form to update with current hardware values

But short enough that users don't notice any delay.

### Complete Solution Summary

```elixir
def mount(_params, _session, socket) do
  # Detect reconnection using official _mounts counter
  is_reconnect = connected?(socket) &&
                 (get_connect_params(socket)["_mounts"] || 0) > 0

  if is_reconnect do
    Process.send_after(self(), :clear_reconnect_flag, 500)
  end

  {:ok, assign(socket, :ignoring_reconnect_events, is_reconnect)}
end

# Guard clause blocks ALL events during reconnection
def handle_event(_event, _params, %{assigns: %{ignoring_reconnect_events: true}} = socket) do
  {:noreply, socket}
end

# Normal event handlers (only run when NOT reconnecting)
def handle_event("update_video", params, socket) do
  # Process normally - this is real user input
  {:noreply, update_settings(socket, params)}
end

def handle_info(:clear_reconnect_flag, socket) do
  {:noreply, assign(socket, :ignoring_reconnect_events, false)}
end
```

### Why This Solution Is Robust

#### ✅ Server-Side Only

- No JavaScript hooks required
- Can't be bypassed by client manipulation
- Works regardless of client-side code

#### ✅ Official LiveView API

- Uses `get_connect_params(socket)["_mounts"]`
- This is LiveView's internal reconnection counter
- Guaranteed to be accurate and maintained

#### ✅ Blocks ALL Events

- Pattern matching catches everything with `_event`
- No need to modify each handler individually
- Add new handlers without worrying about reconnection

#### ✅ Time-Limited

- Automatically clears after 500ms
- No risk of permanently blocking events
- Short enough to be unnoticeable to users

#### ✅ No Race Conditions

- Flag is set synchronously in `mount/3`
- Guaranteed to be set before any events process
- Timer ensures it always clears

#### ✅ Distinguishes First vs. Reconnect

- First connection (`_mounts = 0`): events work immediately
- Reconnections (`_mounts > 0`): events blocked for 500ms
- Best user experience for both scenarios

### Key Lessons Learned

#### 1. LiveView Remounts on Reconnection

When the WebSocket reconnects, `mount/3` is called again from scratch. Your code must handle this gracefully and distinguish between first mount and remount.

#### 2. Use Official APIs When Available

Phoenix provides `get_connect_params(socket)["_mounts"]` specifically for detecting reconnections. Don't reinvent the wheel with custom flags or JavaScript hooks.

#### 3. Pattern Matching Is Powerful

A single guard clause with pattern matching can protect all your event handlers elegantly. This is more maintainable than adding checks to each handler.

#### 4. Queued Events Are Stale

Events queued during disconnection contain old data. Block them temporarily after reconnection and let fresh data arrive from your source of truth (MQTT, database, etc.).

#### 5. Race Conditions Are Real

Client-side hooks and async processes can execute in unpredictable order. Server-side synchronous checks in `mount/3` eliminate race conditions.

#### 6. Test Reconnection Scenarios

Bugs related to reconnection are easy to miss in development. Explicitly test:

- Disconnect WiFi briefly
- Put laptop to sleep
- Simulate network issues with browser dev tools throttling
- Check logs to verify stale data isn't sent

#### 7. Balance Protection vs. UX

- **Too short (< 100ms):** Stale events might sneak through
- **Too long (> 1000ms):** Users notice the delay
- **500ms is a sweet spot** for most applications

### Testing the Fix

To verify this works:

1. Open browser dev tools → Network tab
2. Change a value on your hardware (Phase Horizontal = 50)
3. Throttle network or disconnect WiFi for 2 seconds
4. Change hardware again while disconnected (Phase Horizontal = -25)
5. Reconnect
6. Check MQTT logs → Should NOT see stale value (50) being sent ✅
7. Check browser → Should show current value (-25) ✅

**Expected MQTT Log (Correct):**

```
cy-vp4/videoproc/status/volatile/genlock_phase -25 10  # Fresh value ✅
# No stale "50" being sent back
```

**Before Fix (Incorrect):**

```
cy-vp4/videoproc/status/volatile/genlock_phase -25 10  # Current value
cy-vp4/videoproc/action/set/genlock genlock 50 10      # Stale value! ❌
```

### Real-World Impact

This bug pattern affects any LiveView application where:

- Form data comes from external sources (IoT devices, databases, APIs)
- Data can change while user is disconnected
- Sending stale data has consequences (hardware resets, data corruption, financial transactions)

**Common scenarios:**

- IoT dashboards (our case)
- Collaborative editing tools
- Real-time monitoring systems
- Trading platforms
- Inventory management

### Conclusion

This bug demonstrates how reconnection handling is a critical but often overlooked aspect of real-time applications. The stale data problem can cause:

- Hardware resets
- Data corruption
- User confusion
- Lost work

By using LiveView's official `_mounts` counter and pattern matching to block events during reconnection, I created a robust solution that:

- Requires minimal code (~15 lines)
- Has no race conditions
- Works reliably in all scenarios
- Doesn't degrade user experience
- Is easy to understand and maintain

The final code is simpler, more reliable, and more maintainable than all our attempted workarounds combined. **The key insight: Never trust the client :D**