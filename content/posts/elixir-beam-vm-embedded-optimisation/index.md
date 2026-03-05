---
title: "How We Cut Phoenix Memory from 300MB to 120MB on Embedded"
date: 2026-03-05T10:00:00+01:00
draft: false
comments: true
cover: "/img/elixir-beam-vm-embedded-optimisation/preview.png"
tags:
  - code
  - elixir
  - tutorial

seo:
  - beam vm memory optimization
  - phoenix embedded device
  - erlang allocator tuning
  - elixir low memory
  - beam scheduler tuning
  - phoenix raspberry pi
  - erlang vm.args
  - phoenix memory usage
  - beam allocator flags
  - elixir embedded

seo_description: "Practical guide to reducing BEAM VM memory from 300MB to 120MB and idle CPU from 45% to 5-10% for Phoenix apps on embedded devices. Covers allocator tuning, scheduler limits, PubSub pool sizing, and other production-tested optimisations."
---

At [CyanView](https://cyanview.com), we run a Phoenix LiveView application on embedded hardware. Out of the box, the BEAM VM was eating 300MB of RAM at idle and keeping the CPU at 45%. After tuning, we got it down to ~120MB and 5-10% idle CPU.

Here's exactly what we changed and why each thing matters.

<!--more-->

## Where Does 300MB Go?

Most of that memory isn't actually "used" it's **pre-allocated** by the BEAM's memory allocators. The VM aggressively grabs memory in large carrier blocks for performance on servers with plenty of RAM. On an embedded device, this is wasteful.

You can see the gap yourself:

```elixir
# actual memory used by the VM
:erlang.memory() |> Enum.map(fn {k, v} -> {k, "#{Float.round(v / 1_048_576, 1)} MB"} end)

# with :recon shows allocated vs used
:recon_alloc.memory(:allocated)
:recon_alloc.memory(:used)
```

The difference between allocated and used is what we're clawing back.

## 1. BEAM Allocator Tuning (Biggest Win)

This is the single most impactful change. Add these flags to your `rel/vm.args.eex`:

```bash
## address-order best-fit for all allocators (returns memory to OS faster)
+MBas aobf
+MHas aobf
+MPas aobf
+MDas aobf
+MFas aobf

## disable carrier utilization limit (prevents holding empty carriers)
+MBacul 0
+MHacul 0

## smaller carrier sizes
+MBsmbcs 64
+MBlmbcs 128
+MHsmbcs 64
+MHlmbcs 128

## smaller main carrier
+MMscs 20
```

**What this does:** By default, BEAM allocators use a best-fit strategy optimised for throughput on multi-core servers. They hold onto freed carrier blocks in case they're needed again. The `aobf` (address-order best-fit) strategy is more aggressive about returning memory to the OS. Reducing carrier sizes (`smbcs`, `lmbcs`) means the VM grabs smaller chunks at a time.

This alone dropped our idle memory from ~300MB to ~150MB.

## 2. Reduce Schedulers

BEAM spawns one scheduler per CPU core by default. Each scheduler has its own set of allocator instances, run queues, and timers. On embedded hardware you don't need 4-8 schedulers.

```bash
## single scheduler adjust to your workload
+S 1:1
+SDcpu 1:1
+SDio 1
```

We went with 1 scheduler for our use case. If your app handles concurrent HTTP requests under load, try `+S 2:2` instead. The sweet spot depends on your actual concurrency needs.

**Impact:** This took us from ~150MB down to ~120MB and was the main reason idle CPU dropped from 45% to 5-10%.

## 3. Scheduler Wakeup Tuning

We already had some of these, but the full set matters:

```bash
## don't burn CPU waiting for work
+c false
+sbwt none
+sbwtdcpu none
+sbwtdio none

## low wakeup threshold schedulers sleep faster when idle
+swt very_low
+swtdcpu very_low
+swtdio very_low

## disable scheduler compaction for IO
+secio false

## enable kernel poll (epoll/kqueue)
+K true
```

The `+sbwt none` flags tell schedulers not to busy-wait when there's no work. Without this, each scheduler spins in a tight loop consuming CPU even when idle. The `+swt very_low` flags make schedulers go to sleep faster after a period of no work.

## 4. Resource Limits

Reduce the pre-allocated tables if your app doesn't need the defaults:

```bash
## atom table (default: 1048576)
+t 100000

## max processes (default: 262144)
+P 50000

## max ports (default: 65536)
+Q 8192
```

Each of these tables is pre-allocated at startup. Smaller tables = less baseline memory. Just make sure your app doesn't hit these limits under load.

## 5. Strip Beams in Release

Debug info in `.beam` files adds up. Strip it for production:

```elixir
# mix.exs
def releases() do
  [
    my_app: [
      include_executables_for: [:unix],
      include_erts: true,
      strip_beams: true
    ]
  ]
end
```

This saves a few MB on disk and in memory when modules are loaded.

## 6. PubSub Pool Size

Phoenix PubSub defaults to `System.schedulers_online()` pool workers. On embedded, you only need one:

```elixir
# application.ex
{Phoenix.PubSub, name: MyApp.PubSub, pool_size: 1}
```

## 7. LiveView Socket Tuning

Disable longpoll (saves a separate transport), enable compression, and set a reasonable timeout:

```elixir
# endpoint.ex
socket "/live", Phoenix.LiveView.Socket,
  websocket: [
    connect_info: [session: @session_options],
    compress: true,
    timeout: 60_000
  ]
```

Removing the `longpoll:` option means the server doesn't maintain the longpoll transport at all. If your embedded device only serves local network clients, you likely don't need it.

## 8. Logger Level

Less log buffering in production:

```elixir
# config/prod.exs
config :logger, level: :warning
```

Every log message allocates memory for formatting and buffering. On embedded, you usually want to see warnings and errors only.

## 9. Add :recon for Diagnostics

Add `:recon` as a dependency so you can inspect allocator behaviour at runtime:

```elixir
# mix.exs deps
{:recon, "~> 2.5"}
```

Then in a remote console:

```elixir
# see the gap between allocated and used
:recon_alloc.memory(:allocated)
:recon_alloc.memory(:used)

# per-allocator breakdown
:recon_alloc.memory(:allocated, :current)
```

## Already Using Bandit?

If you're still on Cowboy, consider switching to [Bandit](https://github.com/mtrudel/bandit). It tends to use less memory per connection. Phoenix 1.8 already defaults to Bandit:

```elixir
# config.exs
config :my_app, MyAppWeb.Endpoint,
  adapter: Bandit.PhoenixAdapter
```

## The Full vm.args.eex

Here's our complete file for reference:

```bash
## --- memory optimisation (embedded/low-RAM targets) ---

## address-order best-fit for all allocators
+MBas aobf
+MHas aobf
+MPas aobf
+MDas aobf
+MFas aobf

## disable carrier utilization limit
+MBacul 0
+MHacul 0

## smaller carrier sizes
+MBsmbcs 64
+MBlmbcs 128
+MHsmbcs 64
+MHlmbcs 128

## smaller main carrier
+MMscs 20

## --- scheduler tuning ---

+S 1:1
+SDcpu 1:1
+SDio 1

## --- resource limits ---

+t 100000
+P 50000
+Q 8192

## --- general ---

+c false
+sbwt none
+sbwtdcpu none
+sbwtdio none
+swt very_low
+swtdcpu very_low
+swtdio very_low
+secio false
+K true
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Idle RAM | ~300MB | ~120MB |
| Idle CPU | ~45% | 5-10% |
| Release size | baseline | ~15% smaller (strip_beams) |

The allocator tuning and scheduler reduction account for about 90% of the improvement. Everything else is incremental but worth doing on constrained hardware.

If you're deploying Elixir to Raspberry Pi, industrial gateways, or any embedded Linux target start with the allocator flags. You'll see results immediately.

Check some of me other content: 

- [Why Elixir Is the Smartest Stack Choice](https://1703.lu/blog/why-elixir-is-the-smartest-stack-choice-at-every-stage-of-your-startup)
- [CyanView & Elixir in Broadcasts](https://www.youtube.com/watch?v=CseWOnKuk7o)
