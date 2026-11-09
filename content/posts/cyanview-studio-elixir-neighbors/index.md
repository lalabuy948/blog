---
title: "The BEAM Is a Friendly Host: C, Rust, Svelte and WebGPU Next to Elixir"
date: 2026-10-01T09:00:00+02:00
draft: false
comments: true
cover: "/img/cyanview-studio-elixir-neighbors/preview.webp"
tags:
    - elixir
    - code
    - broadcast

seo:
    - elixir nif vs port
    - dirty nif elixir
    - elixir tauri desktop app
    - burrito elixir cross compile
    - zig cc nif cross compile
    - svelte inside liveview
    - webgpu liveview
    - elixir 3d lut generation
    - mix targets desktop
    - elixir polyglot architecture

seo_description: "How a LiveView dashboard on an ARM32 camera panel became a desktop color studio: Tauri and Burrito, a dirty C NIF that builds a 3D LUT in 0.85 ms per box, one Svelte island inside LiveView, WebGPU scopes, a C++ OpenFX host and Zig cross-compiling. Plus when to use a Port, a NIF, or nothing in the VM at all."
---

A few weeks ago I wrote down my [CyanCamera talk](/posts/cyancamera-turning-phone-into-broadcast-camera/). This is my second talk of the year, from [Goatmire](https://2026.goatmire.com/), this time for an Elixir crowd.

Short version: we had a small LiveView dashboard on a camera control panel. It grew into a desktop color studio. Every time things got harder we put another language next to Elixir, and we never rewrote anything.

<!--more-->

## What's the plan?

- Where it started: a dashboard on an ARM32 panel
- Why color made it hard
- Desktop app with Tauri and Burrito, same Elixir code inside
- The neighbors: C, Svelte, WebGPU, C++, Rust, Zig
- My rule for where foreign code goes

## A dashboard on an ARM32

![CyanView RCP panel](/img/cyanview-studio-elixir-neighbors/rcp-panel.jpg)

The RCP is a physical panel that controls camera heads from many brands. We support 200+ camera models and 30+ control protocols, so hiding that mess from the operator is the whole job.

Embedded people expected C. We picked Elixir on a small ARM board, because the panel can't go down during a show. This is all the VM tuning it needs (long version in [the embedded memory post](/posts/elixir-beam-vm-embedded-optimisation/)):

```
+S 1:1        one scheduler
+P 50000      process limit
+MBas aobf    return memory to the OS fast
```

The dashboard is plain LiveView served by the panel itself. Cameras talk faster than people read, so we keep the latest value per `{camera, event}` and flush every 250 ms.

```elixir
@batch_interval 250

def handle_camera_event(socket, camera_id, event, value) do
  pending = Map.put(socket.assigns.pending_updates, {camera_id, event}, value)
  ...
end
```

![Dashboard controls](/img/cyanview-studio-elixir-neighbors/controls.webp)

Then it grew. A panel has a fixed number of knobs and a camera has hundreds of functions, so everything else went into the dashboard: 73 control components, macros that fire many cameras at once, OSC from show automation like CuePilot, and remote sessions where the operator isn't even in the same building.

## Then color got serious

Our new color box, the iVP, and CyanCamera came with a lot more color science, and operators wanted camera control and looks in one place.

![CyanView iVP](/img/cyanview-studio-elixir-neighbors/ivp.jpg)

![3D LUT cube](/img/cyanview-studio-elixir-neighbors/lut-cube.svg)

What we send to color boxes is a 3D LUT: a cube of 33×33×33 colors, 35,937 entries. Change the look, and you build a new cube and push it to the box. Easy for one box. We have three problems though:

1. Fan-out: 30+ boxes, each with its own chain, each needs a fresh LUT on every slider tick.
2. UI: node graphs, GPU scopes, color wheels. Far from forms.
3. We're not going to rewrite the dashboard.

A single-scheduler ARM board was never going to do that. So we tried a desktop app with the same Elixir app inside.

## Tauri + Burrito

![The holy trinity](/img/cyanview-studio-elixir-neighbors/holy-trinity.svg)

![Desktop architecture](/img/cyanview-studio-elixir-neighbors/desktop-architecture.svg)

[Tauri](https://tauri.app/) is the shell: Rust, native window, menus, updater. [Burrito](https://github.com/burrito-elixir/burrito) packs ERTS and our release into one binary per platform. I wrote the step by step in [Elixir LiveView Single Binary](/posts/elixir-liveview-single-binary/).

Tauri starts that binary as a sidecar, waits until Phoenix answers on localhost and opens the webview.

```rust
handle.shell()
    .sidecar("cy_camera_companion_backend")?
    .env("CY_COMPANION_PORT", "4173")
    .env("BURRITO_TARGET", "tauri")
```

It polls once a second, up to 180 times. Three minutes sounds crazy, but on the first launch on Windows Burrito unpacks the whole release while Defender scans every file. That's all the Rust-to-Elixir integration we have.

The desktop build isn't a fork, it's a Mix target:

```elixir
defp elixirc_paths(_env, :desktop), do: ["lib"]
defp elixirc_paths(_env, _target),
  do: ["lib/cy_camera_companion", "lib/cy_camera_companion_web", ...]
```

And the supervision tree starts a desktop child only if its module was compiled in:

```elixir
|> Enum.flat_map(fn {mod, spec} ->
  if Code.ensure_loaded?(mod), do: List.wrap(spec), else: []
end)
```

The old dashboard routes and the new studio routes live in the same router. "Reuse the dashboard" turned out to mean "leave it where it is".

## The neighbors

| Problem                                | Neighbor      | How                                    |
| :------------------------------------- | :------------ | :------------------------------------- |
| LUTs to 30+ boxes, 0.85 ms each at 33³ | C             | dirty NIF + `Task.async_stream`        |
| Looks change on air without a jump     | C + OTP       | a lerp NIF + one fader GenServer       |
| Colorists think in node graphs         | Svelte        | one island inside LiveView             |
| Scopes on the live grade               | WebGPU        | WGSL compute in the webview            |
| Live video effects at 1080p60          | C++ and Metal | our own OpenFX host, in the Rust shell |
| Three operating systems from one Mac   | Zig and Rust  | cross-compiled NIF inside Burrito      |

### The C NIF, or how a Port lost

Building a LUT touches every entry, 35,937 of them, 30 times per slider tick, 60 ticks a second. Pure Elixir wasn't an option.

| The problem                           | Size                                   |
| :------------------------------------ | :------------------------------------- |
| One 3D LUT at 33³                     | 35,937 RGB triplets, 431 KB of float32 |
| Targets on the network                | 30 plus cameras and color boxes        |
| Each with its own chain               | its own LUT, not a copy                |
| Slider events                         | 60 per second, still moving            |
| Target we set                         | 1 to 2 ms per box                      |
| Measured, 33³, one target end to end  | 0.85 ms                                |

It started as a Port. Then we measured: generation took about 1 ms, and the 20 ms debounce in front of it was pure waiting. So we moved it to a dirty NIF and deleted the coordinator.

```c
static ErlNifFunc nif_funcs[] = {
  {"generate_mixed_nif", 1, generate_mixed_nif, ERL_NIF_DIRTY_JOB_CPU_BOUND},
  {"lerp_lut_nif",       3, lerp_lut_nif,       0},
  {"encode_u16_le_nif",  1, encode_u16_le_nif,  0},
  ...
};
```

| LUT generation               | Measured                                              |
| :--------------------------- | :---------------------------------------------------- |
| Full chain, 17³ / 33³ / 65³  | 0.15 / 0.57 / 3.7 ms                                  |
| First call in a fresh VM     | ~3 ms, so we warm it up at mount                      |
| Color spaces                 | 42, sRGB to ACES to Apple Log                         |
| Parallelism                  | GCD on macOS, OpenMP elsewhere, one row per work item |

The color science is plain C11, built by `elixir_make` during `mix compile`. Elixir packs the grade into one binary and gets back one binary of size³ × 3 floats, no copying.

> ⚠️ A NIF crash takes down the whole BEAM, the Port crash was isolated. So every input is bounds-checked, size is capped before any allocation (`MAX_LUT_SIZE` 129, `MAX_CHAIN` 64), and if the NIF fails to load the app runs degraded and says so. We accept this for one hot path in our own code. Everything else stays in Ports.

The fan-out is a GenServer with one job in flight. A newer request replaces the pending one, so a slider drag never queues up. Every box gets its own task in `Task.async_stream` and pushes its LUT as soon as the NIF returns.

![LUT fan-out](/img/cyanview-studio-elixir-neighbors/lut-fanout.svg)

Funny part: our first version was a batched NIF, one call for all 30 boxes. On paper faster. In practice every box waited for the slowest one, so a DCTL box at 150 ms blocked a simple grading box at 3 ms. We went back to one task per box and let the BEAM schedule them. The revert deleted 120 lines of C and 70 lines of Elixir. Best kind of commit.

| Fan-out wall time, ms | 1 target |    4 |    8 |   16 |
| :-------------------- | -------: | ---: | ---: | ---: |
| 17³                   |     0.42 |  1.0 |  1.5 |  2.7 |
| 33³                   |     0.85 |  2.6 |  4.7 |  9.6 |
| 65³                   |      4.0 | 14.8 | 29.5 | 59.6 |

*Full chain S-Log3 → grading → 3D LUT → Rec.709, Mac with 10 schedulers, 200 runs after warm-up.*

Elixir's part is about 0.25 ms and stays flat. The only slow Elixir step, writing a `.cube` file (5 ms), runs async, off the critical path. My rule of thumb: if it touches every LUT entry on the hot path, it's C. The rest stays in Elixir.

For scale, 30 boxes at 60 Hz in raw bytes:

| LUT size | Frame size | 30 cameras @ 60 Hz    |
| :------- | :--------- | :-------------------- |
| 17       | 78 KB      | 141 MB/s              |
| 33       | 575 KB     | 1,035 MB/s (≈ 1 GB/s) |
| 65       | 4.4 MB     | 7,920 MB/s (≈ 8 GB/s) |

Each device has its own process under a DynamicSupervisor. Binaries over 64 bytes are reference counted, so sending the same LUT to 30 processes copies nothing.

| Device                 | Transport                       | Payload                                             | Encode at 33³ |
| :--------------------- | :------------------------------ | :-------------------------------------------------- | ------------: |
| CyanCamera, the phone  | WebSocket                       | JSON header + RGBA float32, laid out for Metal       |      0.012 ms |
| VP4                    | WebSocket `/ws/look/{ch}`       | 16.16 fixed point, u32 big-endian                   |      0.023 ms |
| AJA ColorBox           | REST PUT, then WebSocket :5000  | 4-byte ASCII slot + uint16 LE triplets              |      0.018 ms |
| AJA FS-HDR             | REST, then WebSocket `/luts`    | same uint16 LE, exactly 215,622 bytes at 33³        |      0.013 ms |
| Flanders BoxIO         | raw TCP, magic `0x42340299`     | 12-bit or resampled to the box's grid, checksummed  |      0.094 ms |
| DaVinci Resolve        | WebSocket to our WsLut plugin   | same 16.16 fixed point, signed                      |  not measured |

Two of those encoders used to be Elixir comprehensions taking ~2 ms, longer than building the LUT itself. In C they're about 100 times faster. And yes, the first row is the phone from the previous post: its Metal shader LUT goes out through the same fan-out as a rack-mounted AJA box.

### Fades on air

A look can't jump on air, every viewer would see it. So any push can crossfade, per device, even while the operator keeps dragging.

| Fade          | Setting                                                      |
| :------------ | :----------------------------------------------------------- |
| Default       | 25 steps over 1 second, up to 199 steps over 60 seconds      |
| One step      | one lerp NIF call: a × (1 − t) + b × t, 0.010 ms at 33³      |
| Push mid-fade | retargets from the last blend sent, no snap                  |
| Last step     | lands on the target LUT bit-exact                            |
| Per node type | choose fade or straight, e.g. a receiver goes straight       |

```elixir
def step_fade(%{step: step, frames: frames} = fade) do
  step = step + 1

  with true <- step < frames,
       {:ok, blend} <- LutNif.lerp_lut(fade.from, fade.to, step / frames) do
    {:fading, %{fade | step: step}, blend}
  else
    false -> {:settled, fade.to}
    {:error, _} -> {:settled, fade.to}
  end
end
```

One GenServer, a map of fades per device, `Process.send_after` as the clock. Ticks are anchored to the fade start, so time spent blending and sending doesn't stretch the fade. 174 lines of Elixir plus one C function, and all four wire formats got fades at once.

<video controls muted playsinline style="max-width: 100%; height: auto;">
  <source src="/img/cyanview-studio-elixir-neighbors/lut-fade.mp4" type="video/mp4">
</video>

### One Svelte component

![Node editor](/img/cyanview-studio-elixir-neighbors/node-editor.jpg)

Colorists want to drag nodes around, not fill forms, and LiveView alone isn't great at graph editors. So there is exactly one Svelte component in the whole app. Nodes and edges go in as props, events come back:

```heex
<.svelte name="LutFlow" id="LutFlow"
  props={%{nodes: @nodes, edges: @edges, chainIds: @chain_ids,
           selectedNodeId: @selected_node_id, rcpActive: @rcp_active}}
  ssr={false} class="absolute inset-0" />
```

The graph is xyflow-shaped JSON everywhere (SQLite, assigns, the client), so no translation layer. The server re-validates every edit: one root, no cycles. And no Node in the build: mix runs bun, bun runs esbuild, esbuild runs the Svelte compiler.

### WebGPU scopes

The colorist needs to see the signal while dragging. That's millions of pixels per frame, so they never go to the server. The server builds the LUT and pushes it, the browser does the rest.

| Scope       | WebGPU pipeline                                                    |
| :---------- | :----------------------------------------------------------------- |
| Histogram   | compute: count → compute: build vertices → render                  |
| Waveform    | render into an `rgba16float` HDR texture, additive blend → tonemap |
| Parade      | same HDR path, per channel                                         |
| Vectorscope | same HDR path, chroma plane                                        |
| Preview     | the server's LUT as a 3D texture, applied in WGSL                  |

I did the WebGL version in [the scopes post](/posts/phoenix-liveview-webgl-color-scopes/), with WebGPU histogram counting moved to compute shaders too.

<video controls muted playsinline style="max-width: 100%; height: auto;">
  <source src="/img/cyanview-studio-elixir-neighbors/webgpu-scopes.mp4" type="video/mp4">
</video>

### OpenFX on live SDI

Next request: OpenFX plugins on the live feed.

| Live video         |  Number |
| :----------------- | ------: |
| Frame rate         | 1080p60 |
| Budget per frame   |  <16 ms |
| One RGBA frame     |  8.3 MB |
| Per second         |  415 MB |

None of that goes through a garbage collected VM.

So we wrote FXVP, our own OpenFX host in C++ with a zero-copy Metal pipeline. Video goes DeckLink → Rust → C++ → SDI and the BEAM never sees a frame. Elixir only sends the chain definition and a few floats. The same engine also runs as a daemon on a headless Mac mini with a DeckLink card, because a laptop has one GPU and a truck has a lot of feeds.

![OFX video path](/img/cyanview-studio-elixir-neighbors/ofx-video-path.svg)

### Cross-compiling from one Mac

Burrito only recompiles NIFs from dependencies, not from your own app. So we added a step that rebuilds `c_src` with `zig cc` for each target:

```elixir
System.cmd("make", ["all", "--always-make", "PRIV_DIR=#{priv_native}"],
  cd: "c_src",
  env: [
    {"CC", "zig cc -target #{triplet}"},
    {"ERTS_INCLUDE_DIR", erts_include}
  ])
```

Catch: zig doesn't ship libomp, so cross-built LUT generation is single-threaded. On Windows `load_nif` wants `.dll`, so we rename the `.so`. It all runs as one docker build on a Mac.

### Licensing

Seats are per laptop. First version used hostname + MAC address. Then someone plugs into a dock, or joins a VPN, the MAC changes, and one laptop eats several seats 🙃

Now we take the machine GUID the OS already has (`IOPlatformUUID`, `MachineGuid`, `machine-id`) and use it as the key for an HMAC over the app name. Stable per laptop, and the GUID never leaves the machine. The server returns a signed token that the app checks offline.

A truck in a stadium car park is offline more often than online, so a failed heartbeat doesn't lock anything. You get 72 hours of grace. A revoked seat applies right away.

## Where foreign code goes

| Where          | When                                                          | Examples                                            |
| :------------- | :------------------------------------------------------------ | :-------------------------------------------------- |
| Port           | by default: someone else's code, or anything that can crash   | user DCTL shaders, ffmpeg, ffprobe, the C++ compiler |
| NIF            | after a benchmark, our own code, every input bounds-checked   | LUT generation and fades                            |
| Outside the VM | frames, pixels, shaders                                       | OFX host, DeckLink, NDI, WebGPU scopes              |

If you go the same way:

- Start with a Port. Move to a NIF only when a benchmark says so.
- Keep pixels out of the VM, keep decisions in it.
- One process per device, so one broken protocol doesn't kill the rest.
- Mix targets instead of forks. We have one codebase the whole time and I'd do it again.

Elixir is in every part of this, but most of the time it's the boring part. I like it that way.

Feel free to leave a comment.

## Resources

- [Tauri](https://tauri.app/)
- [Burrito](https://github.com/burrito-elixir/burrito)
- [Elixir LiveView Single Binary](/posts/elixir-liveview-single-binary/)
- [CyanCamera: Turning an iPhone Into a Broadcast Camera](/posts/cyancamera-turning-phone-into-broadcast-camera/)
- [Broadcast Grade Color Scopes in Phoenix LiveView](/posts/phoenix-liveview-webgl-color-scopes/)
- [How We Cut Phoenix Memory from 300MB to 120MB on Embedded](/posts/elixir-beam-vm-embedded-optimisation/)
- [CyanView](https://www.cyanview.com/)
- [1703 Group | Elixir Agency](https://1703.lu/)
