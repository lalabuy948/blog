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

seo_description: "How a LiveView dashboard on an ARM32 camera panel became a color studio on three operating systems: Tauri and Burrito for the shell, a dirty C NIF for 3D LUT fan-out at 0.85 ms per box, Svelte as one island inside LiveView, WebGPU scopes, a C++ OpenFX host in Rust, and Zig to cross-compile it all from one Mac. Plus the rule for when foreign code goes in a Port, a NIF, or outside the VM."
---

A few weeks ago I wrote down my talk on [how one phone becomes a broadcast camera](/posts/cyancamera-turning-phone-into-broadcast-camera/). This is the other end of that cable, from the second talk I gave this year, this time to an Elixir crowd at [Goatmire](https://2026.goatmire.com/).

I am not going to pitch Elixir. I am going to tell you how a small dashboard on a camera control panel turned into a color studio, and how every time the problem got harder, we added a neighbor language next to Elixir instead of rewriting.

Elixir is in every part of this story. It is the star in none of them. That is the point.

<!--more-->

Quick context for anyone new here. I own the software side of [CyanView](https://www.cyanview.com/): the dashboard that speaks to every camera in the truck, the web studio with [broadcast-grade scopes](/posts/phoenix-liveview-webgl-color-scopes/), and the CyanCamera app. Before CyanView it was zero-downtime migrations and systems serving millions of users, a lot of it in Elixir.

## Previously, on CyanCamera

![Pro pipeline](/img/cyanview-studio-elixir-neighbors/pro-pipeline.svg)

The phone post ended with a 33-point cube running in a Metal compute shader, converting Apple Log to Rec.709 live. This is the same pipeline, drawn for a whole truck.

Light hits the sensor. Camera control shapes what the sensor captures: iris, ND, shutter, gain, white balance. The camera does not send a pretty picture, it sends log: fourteen stops folded into ten bits, flat and grey on purpose, over SDI. You never watch log, you convert log. The conversion is a 3D LUT on a color box, and it carries the look at the same time. Then the vision mixer, then on air.

The phone was one camera and a LUT applied on the device. Today is: who builds that LUT, and how it reaches thirty boxes while the operator is still dragging the slider. The phone was Swift and Metal. Today is Elixir, with C, C++, Zig, Rust, Svelte and WebGPU next to it.

For years the two brackets on that diagram were two tools, often two operators. The bar on top, one grade driving both sides, is what this post is about.

## Every broadcast is a zoo

| The zoo          | What it looks like                                                     |
| :--------------- | :--------------------------------------------------------------------- |
| Cameras          | 200+ models, 30+ different control protocols                           |
| Where they are   | in the truck, across the venue, or remote over the internet            |
| Video            | SDI, SRT, NDI, HDMI …                                                  |
| Color boxes      | Flanders BoxIO, AJA ColorBox, AJA FS-HDR, VP4, and our new iVP         |
| And now virtual  | color in software: DaVinci Resolve, OFX hosts, a virtual VP4 in the app |
| In front of it   | one RCP panel, one feel, whatever the brand                            |

Trucks, galleries and remote control rooms run gear from dozens of vendors that agree on nothing. Video arrives over SDI, SRT, NDI, HDMI. Some cameras are in the truck, some across the stadium, some on another continent. Then the look: color boxes from Flanders, AJA, our own VP4 and now the iVP. And the newest member of the zoo is not a box at all. Color in software, which is where this is going.

Operators do not care about any of that. They want one panel, one feel, one source of truth. The RCP's job is to hide the zoo behind a calm interface.

## Act one: a dashboard on an ARM32

![CyanView RCP panel](/img/cyanview-studio-elixir-neighbors/rcp-panel.jpg)

The RCP is a physical panel. One device, many camera heads, one consistent feel across brands. The embedded world expected C. We picked Elixir, on a small ARM board. Uptime is the contract, not a feature.

```
+S 1:1        one scheduler
+P 50000      process limit
+MBas aobf    return memory to the OS fast
```

That is the entire VM tuning for the panel (the long version is in [the embedded memory post](/posts/elixir-beam-vm-embedded-optimisation/)). Supervision trees and isolated processes are why operators trusted it.

The dashboard started small: aggregate values from all cameras in one browser tab, served by the panel itself. Cameras talk faster than people read, so we keep the latest value per `{camera, event}` and flush every 250 ms.

```elixir
@batch_interval 250

def handle_camera_event(socket, camera_id, event, value) do
  pending = Map.put(socket.assigns.pending_updates, {camera_id, event}, value)
  ...
end
```

![Dashboard controls](/img/cyanview-studio-elixir-neighbors/controls.webp)

All of it plain LiveView on the panel's own Phoenix endpoint. The server already knows the truth, so it renders the UI.

Then it grew. The RCP panel has a fixed number of knobs. Cameras have hundreds of functions. Everything that did not fit on the panel found a home in the dashboard:

| Beyond the panel        | What it means                                              | In the app                                                    |
| :---------------------- | :--------------------------------------------------------- | :------------------------------------------------------------ |
| Custom camera functions | every function a camera has, not only the ones with a knob | 73 control components: PTZ, lens, user matrix, color warper … |
| New ways to control     | touch, mouse, keyboard, any browser on the network         | dashboard and control template editors, per show              |
| Macros                  | one button, many cameras or groups at once                 | macro editor, macro pad, raw MQTT escape hatch                |
| Automation              | the rundown drives the cameras                             | OSC in, e.g. CuePilot: `/macro /camera /grade /lut`           |
| Remote production       | operators far from the truck                               | REMI sessions, one per scene                                  |

The dashboard stopped waiting for a person. Macros fire many cameras at once, and show automation like CuePilot sends OSC at the right moment in the rundown. And it went further than the truck: remote productions, where the operator is not in the same building as the cameras.

This was the first horizon. Software, not hardware.

## Act two: then color got serious

The iVP, our openGear successor to the VP4, and CyanCamera put a lot more color science on the table. Operators wanted one interface for camera values and for looks.

![CyanView VP4](/img/cyanview-studio-elixir-neighbors/vp4.jpg)
*CyanView VP4*

![CyanView iVP](/img/cyanview-studio-elixir-neighbors/ivp.jpg)
*CyanView iVP, and a lot of other vendors' boxes next to them*

### What is a 3D LUT

![3D LUT cube](/img/cyanview-studio-elixir-neighbors/lut-cube.svg)

One minute on what we are actually shipping to those boxes, because the rest of the post is about building it fast.

Take every color a pixel can have and lay it out in space: red along one axis, green along the second, blue along the third. Black where the axes meet, white in the opposite corner. A pixel from the camera, log, flat and grey, is 0.42 along red, 0.35 along green, 0.30 up blue. Its three numbers are not a color any more. They are an address.

A 3D LUT stores a new color at regular points in that cube. 33 points per axis, so 35,937 entries. The pixel lands inside one small cell, a cell has eight corners, read the eight stored colors and blend them by distance. The blend is the pixel out. Skin, sky, grass, all go through the same lookup and come out graded.

No if, no branches, no color science at runtime. The conversion to Rec.709 and the creative look are both baked into the same table. That is why a color box can do it for two million pixels, fifty times a second. And it is why changing the look means one thing: build a new table and get it to the box.

### Three problems arrived at once

| Problem                        | Why it is hard                                                              |
| :----------------------------- | :-------------------------------------------------------------------------- |
| LUT fan-out to 30+ color boxes | a fresh 3D LUT per box, per slider tick, every box with its own chain       |
| A heavy interface              | node graphs, GPU scopes, color wheels, curves: far beyond forms             |
| Reuse the dashboard            | 73 camera controls, MQTT plumbing, camera state: we were not rewriting that |

And the panel's ARM board with one scheduler was never the place for node graphs, GPU scopes and thirty LUTs a tick.

So we tried a spike. A desktop app, with the same Elixir app inside.

## Act three: the spike, Tauri + Burrito

![The holy trinity](/img/cyanview-studio-elixir-neighbors/holy-trinity.svg)

Camera control is where CyanView started: iris, gain, ND, white balance, knee. Color boxes are where the look lands: a 3D LUT per feed, in an AJA box, a BoxIO, a VP4 or iVP, or on the phone. For years those were two worlds, two tools, often two operators. The third piece is new, and it is software: one place where a grade drives the sensor and the LUT together.

<video controls muted playsinline style="max-width: 100%; height: auto;">
  <source src="/img/cyanview-studio-elixir-neighbors/studio-blender.mp4" type="video/mp4">
</video>

### One binary, two worlds inside

![Desktop architecture](/img/cyanview-studio-elixir-neighbors/desktop-architecture.svg)

The shell is [Tauri](https://tauri.app/). Rust, a real native window, menus, updater, tray icon. [Burrito](https://github.com/burrito-elixir/burrito) packs ERTS and our release into one binary per platform. I wrote the step-by-step of this setup in [Elixir LiveView Single Binary](/posts/elixir-liveview-single-binary/), so here is just the shape of it.

Tauri launches that binary as a sidecar, sets the target and a port, then polls localhost until Phoenix answers. Then the webview loads the LiveView.

```rust
handle.shell()
    .sidecar("cy_camera_companion_backend")?
    .env("CY_COMPANION_PORT", "4173")
    .env("BURRITO_TARGET", "tauri")
```

Then poll `http://127.0.0.1:4173` once a second, up to 180 times, then open the webview on `/desktop`. Backend stdout goes to `backend.log`, because the sidecar has no console on Windows. That is the entire Rust to Elixir integration. Spawn, poll, load a URL.

Up to three minutes of polling, because first launch on Windows unpacks the whole release under Defender scanning.

The spike went well. The user sees one app. We see the RCP engine wearing a different coat.

### One flag, two products

The desktop build is not a fork. It is a Mix target. The default compile path ships the RCP modules. Flip one variable and the desktop directory compiles in.

```elixir
defp elixirc_paths(_env, :desktop), do: ["lib"]
defp elixirc_paths(_env, _target),
  do: ["lib/cy_camera_companion", "lib/cy_camera_companion_web", ...]
```

The supervision tree does not read a config flag. Each desktop child is guarded by "is this module loaded?". Code that was not compiled in is simply not started.

```elixir
# {gate, child_spec}: start only if the module was compiled in
{Desktop.Services.MqttWorkerSupervisor, Desktop.Services.MqttWorkerSupervisor},
{nil, {Registry, keys: :unique, name: Desktop.Discovery.Registry}},
...
|> Enum.flat_map(fn {mod, spec} ->
  if Code.ensure_loaded?(mod), do: List.wrap(spec), else: []
end)
```

### The dashboard came along for free

```elixir
scope "/", CyCameraCompanionWeb do
  live "/", HomeLive                                   # the RCP dashboard
  live "/scene/:scene_id/camera/:camera_id", CameraControlLive
end

scope "/desktop", CyCameraCompanionWeb do
  live "/projects/:project_id", StudioLive             # the new studio
  live "/projects/:project_id/feeds/:feed_id/scopes", ScopesFullscreenLive
end
```

Same router, same endpoint, same MQTT workers, same camera state. The studio did not import the dashboard. It lives next to it, in the same app. Problem three, reuse the dashboard, was solved by not moving it.

## Act four: then we kept adding neighbors

Once the spike worked, every hard problem got the same treatment. Find the right tool, put it next to Elixir, keep the decisions in Elixir.

| Problem                                | Neighbor      | Mechanism                              |
| :------------------------------------- | :------------ | :------------------------------------- |
| LUTs to 30+ boxes, 0.85 ms each at 33³ | C             | dirty NIF + `Task.async_stream`        |
| Looks change on air without a jump     | C + OTP       | a lerp NIF + one fader GenServer       |
| Colorists think in node graphs         | Svelte        | one island inside LiveView             |
| Scopes on the live grade               | WebGPU        | WGSL compute in the webview            |
| Users bring their own shaders          | C++           | compiled at runtime, run as a Port     |
| Live video effects at 1080p60          | C++ and Metal | our own OpenFX host, in the Rust shell |
| Three operating systems from one Mac   | Zig and Rust  | cross-compiled NIF inside Burrito      |

The answer is not syntax. The BEAM is willing to live next to other ecosystems. It hosts them. It does not try to absorb them.

### WebGPU scopes

The colorist needs to see what the grade does to the signal, while dragging. That is millions of pixels per frame, so it never touches the server. The server does the maths and pushes the LUT. The browser does every pixel.

| Scope       | WebGPU pipeline                                                   |
| :---------- | :---------------------------------------------------------------- |
| Histogram   | compute: count → compute: build vertices → render                 |
| Waveform    | render into an `rgba16float` HDR texture, additive blend → tonemap |
| Parade      | same HDR path, per channel                                        |
| Vectorscope | same HDR path, chroma plane                                       |
| Preview     | the server's LUT as a 3D texture, applied in WGSL                 |

<video controls muted playsinline style="max-width: 100%; height: auto;">
  <source src="/img/cyanview-studio-elixir-neighbors/webgpu-scopes.mp4" type="video/mp4">
</video>

Scopes also open in their own fullscreen window, one route in the same router. The WebGL version of this story is in [the scopes post](/posts/phoenix-liveview-webgl-color-scopes/). WebGPU gave us compute shaders, so the histogram counting moved off the CPU too.

### A heavy interface: Svelte, as one island

![Node editor](/img/cyanview-studio-elixir-neighbors/node-editor.jpg)

Problem two. A grade is a chain: input transform → grading → DCTL → cube file → output transform, groups of all that, linked or copied. Nobody wants to build this in a form. They want to drag boxes. LiveView alone does not do a drag and drop graph editor well.

So, one Svelte component in the whole app. It gets nodes and edges as props. It pushes nine events back.

```heex
<.svelte name="LutFlow" id="LutFlow"
  props={%{nodes: @nodes, edges: @edges, chainIds: @chain_ids,
           selectedNodeId: @selected_node_id, rcpActive: @rcp_active}}
  ssr={false} class="absolute inset-0" />
```

```js
// LutFlow.svelte
props.live.pushEvent("edge_added", {id, source, target})
```

```elixir
def handle_event("edge_added", %{"source" => s, "target" => t}, socket) do
  # re-validate: one root only, no cycles; then save (200 ms debounce),
  # regenerate preview LUT (17³, max one per 16 ms), push it to WebGPU
end
```

The graph is xyflow-shaped JSON. It is stored as that JSON in SQLite, held as that JSON in assigns, and rendered as that JSON by xyflow. No translation layer. Elixir does the topological sort and sends the chain order back down as badges.

A few things that mattered in practice:

- Fixed component id, so xyflow's viewport and undo survive grade switches.
- The server re-validates every client edit. The client validates too, for feel.
- Build: mix runs bun, bun runs esbuild, esbuild runs the Svelte compiler. No Node.

### Fan-out: a grade must reach every box, now

| The problem                           | Size                                   |
| :------------------------------------ | :------------------------------------- |
| One 3D LUT at 33³                     | 35,937 RGB triplets, 431 KB of float32 |
| Targets on the network                | 30 plus cameras and color boxes        |
| Each with its own chain               | its own LUT, not a copy                |
| Slider events                         | 60 per second, still moving            |
| Target we set                         | 1 to 2 ms per box                      |
| Measured, 33³, one target end to end  | 0.85 ms                                |

On the phone this cube ran in a shader. Here we build it, thirty times, per slider tick. An operator drags a slider on a laptop. Every camera and color box on the venue network needs a fresh LUT. Not the same LUT, their own. And the slider is still moving.

Generating one per box per slider tick, in Elixir, is not going to happen.

### C, as a dirty NIF

The one place in the app where C runs inside the VM.

```c
static ErlNifFunc nif_funcs[] = {
  {"generate_mixed_nif", 1, generate_mixed_nif, ERL_NIF_DIRTY_JOB_CPU_BOUND},
  {"lerp_lut_nif",       3, lerp_lut_nif,       0},
  {"encode_u16_le_nif",  1, encode_u16_le_nif,  0},
  ...
};
```

| LUT generation               | Measured                                            |
| :--------------------------- | :-------------------------------------------------- |
| Full chain, 17³ / 33³ / 65³  | 0.15 / 0.57 / 3.7 ms                                |
| First call in a fresh VM     | ~3 ms, so we warm it up at mount                    |
| Color spaces                 | 42, sRGB to ACES to Apple Log                       |
| Parallelism                  | GCD on macOS, OpenMP elsewhere, one row per work item |

The color science is plain C11, built by `elixir_make` during `mix compile`. CDL, curves, primaries, color space conversion. Elixir packs the grade into one binary with native-endian bitstrings and gets back one binary of size³ × 3 floats. No copies.

It used to be a Port. We measured generation at one millisecond and found the twenty millisecond debounce in front of it was pure added latency. We moved to a NIF and deleted the coordinator.

The trade: a NIF crash takes the VM. The comment at the top of the C file says exactly that: "a NIF crash takes down the whole BEAM (the old port crash was isolated). Every `NEED()` bounds check from the former decoders is preserved, and size is capped before any allocation." `MAX_LUT_SIZE` is 129, `MAX_CHAIN` is 64 nodes. A failed load never fails `@on_load`; the app runs degraded and says so.

We took that trade for one hot path, our own code. Everywhere else, the answer is a Port. Hold that thought.

### The fan-out itself

![LUT fan-out](/img/cyanview-studio-elixir-neighbors/lut-fanout.svg)

One GenServer holds one job in flight. A newer request replaces a pending one, so a slider drag never piles up and cameras converge to the last value.

Each box gets a task under `Task.async_stream` with `max_concurrency` set to `schedulers_online`, unordered, killed on timeout. Resolve its chain, call the NIF, push to the device. Each task ships its LUT the instant its NIF returns, no barrier between boxes.

We tried a batched NIF first. It looked faster on paper: one call instead of thirty. In practice every box waited for the slowest LUT, and a DCTL box at 150 ms blocked a grading box at 3 ms. GCD nests cleanly across concurrent NIF calls, so per-box tasks won. The revert removed 120 lines of C and 70 of Elixir. The BEAM schedules.

### Measured

![Measured pipeline](/img/cyanview-studio-elixir-neighbors/measured-pipeline.svg)

Full chain: IDT S-Log3 → grading → primaries → 3D LUT → ODT Rec.709. 10 warm-up calls, then 200 timed, average in ms, on a Mac with 10 schedulers.

| Fan-out wall time, ms | 1 target |    4 |    8 |   16 |
| :-------------------- | -------: | ---: | ---: | ---: |
| 17³                   |     0.42 |  1.0 |  1.5 |  2.7 |
| 33³                   |     0.85 |  2.6 |  4.7 |  9.6 |
| 65³                   |      4.0 | 14.8 | 29.5 | 59.6 |

The fan-out scales with scheduler count, not with target count. Elixir's share of the critical path is a quarter of a millisecond and does not grow with the cube. Every device encoder is a C NIF: 1 to 9 percent of the generate cost.

The one Elixir step that is slow, writing a `.cube` file at 5 ms, runs as an async task off the critical path. That is the rule: anything that touches every LUT entry on the hot path is C. Per-target bookkeeping, and anything off the critical path, stays Elixir.

For scale, this is what thirty boxes at 60 Hz would mean in raw bytes:

| LUT size | Frame size | 30 cameras @ 60 Hz    |
| :------- | :--------- | :-------------------- |
| 17       | 78 KB      | 141 MB/s              |
| 33       | 575 KB     | 1,035 MB/s (≈ 1 GB/s) |
| 65       | 4.4 MB     | 7,920 MB/s (≈ 8 GB/s) |

### Six targets, four wire formats

| Device                 | Transport                       | Payload                                             | Encode at 33³ |
| :--------------------- | :------------------------------ | :-------------------------------------------------- | ------------: |
| CyanCamera, the phone  | WebSocket                       | JSON header + RGBA float32, laid out for Metal       |      0.012 ms |
| VP4                    | WebSocket `/ws/look/{ch}`       | 16.16 fixed point, u32 big-endian                   |      0.023 ms |
| AJA ColorBox           | REST PUT, then WebSocket :5000  | 4-byte ASCII slot + uint16 LE triplets              |      0.018 ms |
| AJA FS-HDR             | REST, then WebSocket `/luts`    | same uint16 LE, exactly 215,622 bytes at 33³        |      0.013 ms |
| Flanders BoxIO         | raw TCP, magic `0x42340299`     | 12-bit or resampled to the box's grid, checksummed  |      0.094 ms |
| DaVinci Resolve        | WebSocket to our WsLut plugin   | same 16.16 fixed point, signed                      |  not measured |

One DynamicSupervisor, one process per device, one Registry. The NIF binary is passed to each device process by cast. Binaries over sixty four bytes are reference counted, so thirty casts is zero copies.

Six targets, but only four wire formats: RGBA floats for the phone, 16.16 fixed point for VP4 and Resolve, uint16 for both AJA boxes, and BoxIO's own frames.

The last column is the benchmark. Until recently two of those encoders were Elixir comprehensions at about two milliseconds, more than generating the LUT itself. Same job in C, a hundred times cheaper.

The first row is the phone from the previous post. The LUT it applies in its Metal shader is built here and shipped by the same fan-out as a rack-mounted AJA box.

Elixir did not make the protocols nicer. It made having six of them boring.

### The live fade

On air, a look must never jump. Changing a look with a snap is visible to every viewer. So every LUT push can crossfade, per feed, per device, while the operator keeps dragging.

<video controls muted playsinline style="max-width: 100%; height: auto;">
  <source src="/img/cyanview-studio-elixir-neighbors/lut-fade.mp4" type="video/mp4">
</video>

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
    false -> {:settled, fade.to}          # last step: the exact target, not a blend
    {:error, _} -> {:settled, fade.to}    # a failed blend snaps, never drops
  end
end
```

One GenServer, a map of fades keyed by device, `Process.send_after` as the clock. Ticks are anchored to the fade start, so blend and send time never stretch the fade. A fresh tick ref per retarget orphans the old timer, so there is no cancel bookkeeping. One device's failed send is rescued, so it never stops every other feed's fade.

174 lines of Elixir plus one C function, in front of every device send. All four wire formats got fades at once.

### Real-time effects on live SDI

The colorist wants OpenFX plugins on the live feed.

| Live video         |                              Number |
| :----------------- | ----------------------------------: |
| Frame rate         |                             1080p60 |
| Budget per frame   |                              <16 ms |
| One RGBA frame     |                              8.3 MB |
| Per second         |                              415 MB |
| Where that belongs | nowhere near a garbage collected VM |

### Rust, on the outside

| Rust does                                      | Rust does not          |
| :--------------------------------------------- | :--------------------- |
| menus, tray, window state, updater             | talk to Elixir directly |
| load the NDI SDK at runtime with `libloading`  | own any business logic |
| DeckLink capture and output threads            | touch the database     |
| bind the C++ OFX host                          | render the UI          |

Rust touches Elixir three times: spawn, poll, load URL. After that the browser is the bridge: LiveView socket one way, Tauri `invoke` the other. There is no Rust to Elixir channel and we never needed one.

### FXVP: our OpenFX engine

![FXVP headless](/img/cyanview-studio-elixir-neighbors/fxvp-headless.svg)

FXVP is the OpenFX engine we wrote for this. The first version runs inside the studio: the Rust shell hosts it, and the panel talks to it with a Tauri invoke.

But a laptop has one DeckLink and one GPU. A truck has many feeds. So the same engine also ships as a daemon on a headless Mac mini with a DeckLink card and no screen. One daemon, one pipeline, one port. It boots, restores its last pipeline, and starts processing before anyone connects to it.

From the operator's seat nothing changes. Same panel, same node list, same sliders. Elixir keeps one client per box, pushes the chain over HTTP, and relays the box's event stream back into the panel over PubSub. Video never crosses the network, only the definition and a few floats.

![OFX video path](/img/cyanview-studio-elixir-neighbors/ofx-video-path.svg)

We wrote the host in C++ on the official HostSupport library, with a zero copy Metal pipeline on Apple Silicon. Tauri's build script compiles it with cmake and links it statically. Video goes DeckLink to Rust to C++ to SDI out. The BEAM never sees a frame.

### Ship to three operating systems from one Mac

![Build pipeline](/img/cyanview-studio-elixir-neighbors/build-pipeline.svg)

Burrito's own build step only recompiles NIFs of dependencies, not your app. So there is one patch step: rebuild `c_src` with `zig cc` against the target's ERTS headers.

```elixir
System.cmd("make", ["all", "--always-make", "PRIV_DIR=#{priv_native}"],
  cd: "c_src",
  env: [
    {"CC", "zig cc -target #{triplet}"},
    {"ERTS_INCLUDE_DIR", erts_include}
  ])

if context.target.os == :windows do
  File.rename!("lut_nif.so", "lut_nif.dll")   # load_nif resolves <path>.dll
end
```

Trade: zig ships no libomp, so cross-built LUTs run single-threaded. Windows is built from Linux with cargo-xwin. The Linux stage runs Ubuntu 22 for an older glibc. Everything is one docker build from a Mac.

### Auto-update

![Auto update](/img/cyanview-studio-elixir-neighbors/auto-update.svg)

Three seconds after launch, the webview asks Tauri's updater plugin to check. The plugin calls our server with the target, the architecture and the current version. If there is something newer, it gets back a version, release notes, a download URL and a signature.

The user sees a small banner: update available, later or install. Install downloads the bundle with a progress bar, and the plugin verifies the signature against the minisign public key compiled into the app. A bundle we did not sign does not install.

Then one Rust command: kill the BEAM sidecar, restart the app. The new bundle boots like any first launch. Burrito unpacks the new release, Phoenix answers on localhost, the webview loads. The shell and the sidecar ship in the same bundle, so they can never drift apart.

Elixir is not involved at any point. No hot code upgrade, no appup files. The release is cargo inside a signed bundle, and for a desktop app that is exactly what we wanted.

### Licensing: a seat is a laptop, so what is a laptop?

![Licence server](/img/cyanview-studio-elixir-neighbors/licence-server.svg)

A licence is sold per seat, and a seat is one laptop. We started with hostname plus MAC address. It changed every time someone plugged into a dock, joined a VPN or renamed the machine, and one laptop ended up holding several seats.

Now we read the machine GUID the OS already keeps: `IOPlatformUUID` on macOS, `MachineGuid` in the Windows registry, `machine-id` on Linux. We never send it. We use it as the key of an HMAC-SHA256 over the app name. The result is 64 hex characters: stable for this laptop, useless to any other app, and not reversible to the GUID.

The rest is one GenServer. Activate once, heartbeat every five minutes, the HTTP call in a Task so the GenServer keeps answering the UI while the network is slow.

And the network will be slow, or gone. A truck in a stadium car park is offline more often than not. So a failed heartbeat locks nothing. The licence goes into grace, 72 hours from the last good heartbeat, and one good heartbeat brings it back. A revoked seat or a deactivated key is different: the server says so, and that is immediate.

## The rule for foreign code

| Where          | When                                                          | Examples here                                       |
| :------------- | :------------------------------------------------------------ | :-------------------------------------------------- |
| Port           | by default: someone else's code, or anything that can crash   | user DCTL shaders, ffmpeg, ffprobe, the C++ compiler |
| NIF            | after a measurement, our own code, every input bounds-checked | LUT generation and fades                            |
| Outside the VM | frames, pixels, shaders                                       | OFX host, DeckLink, NDI, WebGPU scopes              |

User shaders, ffmpeg, compilers: outside the VM, behind a Port or `System.cmd`. Isolated, supervised, restartable. Our own color math, bounds checked, hot path, measured: a NIF. Video frames and scope pixels: not in the VM at all.

## What it bought us

|              | Where we started                | Now                                            |
| :----------- | :------------------------------ | :--------------------------------------------- |
| Company      | hardware and camera control     | plus color science and software                |
| Interface    | a camera dashboard on the panel | the dashboard, plus a color studio on 3 OS     |
| Color        | camera values                   | node graphs, DCTL, OFX, GPU scopes, live fades |
| LUT delivery | some                            | 30+ boxes and a phone, 0.85 ms per box at 33³  |
| Source trees | one app                         | still one app                                  |
| Team         | small                           | still small                                    |

We started with a dashboard that showed camera values. Today the same app also ships a color studio on three operating systems. The dashboard is still there, in the same router, untouched.

The whole argument in one line: with Elixir, the code that joins C, C++, Rust, Svelte and WebGPU stayed small enough that we spent our time on color, not on integration.

## If you are considering the same path

1. Pick the BEAM for anything that has to stay up and talk to strangers.
2. Do not fight other ecosystems. Wrap them. Ports first, NIFs after a benchmark.
3. Keep frames, pixels and shaders out of the VM. Keep the decisions in it.
4. One process per device. Let each protocol fail alone.
5. One codebase, many targets. Mix targets beat forks every time.

The phone earned its place in the rundown. The software earned its place next to the hardware.

## Resources

- [Tauri](https://tauri.app/)
- [Burrito](https://github.com/burrito-elixir/burrito)
- [Elixir LiveView Single Binary: building a desktop app with Tauri and Burrito](https://mrpopov.com/posts/elixir-liveview-single-binary/)
- [CyanCamera: Turning an iPhone Into a Broadcast Camera](/posts/cyancamera-turning-phone-into-broadcast-camera/)
- [Broadcast Grade Color Scopes in Phoenix LiveView](/posts/phoenix-liveview-webgl-color-scopes/)
- [How We Cut Phoenix Memory from 300MB to 120MB on Embedded](/posts/elixir-beam-vm-embedded-optimisation/)
- [CyanView](https://www.cyanview.com/)
- [1703 Group | Elixir Agency](https://1703.lu/)
