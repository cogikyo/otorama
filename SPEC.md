# otorama spec

otorama is a Wayland-native music visualizer: shader overlays that react to system audio, drawn on layer-shell surfaces.
The name joins 音 _oto_ (sound) and Greek ὅραμα _hórama_ (sight), so it reads as "seeing sound".
It replaces GLava in the author's [dotfiles](https://github.com/cogikyo/dotfiles), and it is meant to be released and maintained as a package.

This file holds the whole plan as of 2026-10-11, so a new session can continue without the earlier conversation.
Settled decisions stay settled unless new evidence contradicts them.
Open questions are near the end.

## Status

- The repository holds only a Cargo skeleton; `main.rs` prints hello world.
- No spike, design code, or dependencies exist yet.
- The crates.io name and the AUR package are not reserved; both were free on 2026-10-11.
- `~/dotfiles/packages/repos.lst` clones this repository to `~/otorama`.

## Goals

- Look at least as smooth as GLava, with much less delay between sound and picture.
- Show a correct spectrum: real FFTs, log-spaced bands, a decibel scale, and real bass resolution.
- Make shaders and settings easy to edit live, with hot reload.
- Run on any compositor with wlr-layer-shell, transparent and click-through, on every output.
- Stop drawing when nothing moves.

X11 support and compatibility with GLava shaders or its `#request` config are out of scope.

## Settled decisions

| Topic         | Decision                                                 |
| ------------- | -------------------------------------------------------- |
| Language      | Rust, edition 2024 (newest stable edition on rustc 1.99) |
| License       | Unlicense only                                           |
| Repository    | `cogikyo/otorama`, branch `master`                       |
| Versions      | Stay 0.x; config and shader inputs may break             |
| Compositors   | Any wlr-layer-shell compositor; tested on Hyprland only  |
| Hyprland      | Extras are optional, never required                      |
| Audio         | Native PipeWire capture of the default sink monitor      |
| Starting code | New code; do not fork GLava or vibe                      |
| Publishing    | crates.io and AUR wait for the first beta                |

The config format and the shader inputs are public API.
Version them from the first release, and change them when users report problems.

vibe is AGPLv3, so do not copy its code into this Unlicense project; reading it for ideas is fine.

## Why a new project

### GLava

GLava is abandoned and X11-only.

- The last tag is v1.6.3 (2019-03-12) and the last commit is `6fc0e32` (2021-06-17).
- The author declined Wayland in [issue #27](https://github.com/jarcode-foss/glava/issues/27); it runs under XWayland today.
- About 1,300 lines are X11/GLX window code.

Most of its smoothness comes from delay.
With the author's settings, a 4096-sample window at 22050 Hz covers 186 ms, so its midpoint is about 93 ms old.
Averaging over 6 frames adds about 42 ms, for about 135 ms before PulseAudio buffering.

Its DSP has defects that the smoothing hides:

- The FFT treats neighbouring real samples as complex pairs, takes `abs` of each part, and applies `ln(1+x)/3` without normalization.
- `fftcutoff` is a gain-ramp offset, not a frequency cutoff.
- The average divides by the frame count, not by the sum of the weights.
- The `window`, `gravity`, and `avg` requests do nothing; `fft` applies all of them.
- The CPU FFT runs again each frame on data it already transformed.
- Results go to `GL_R16` textures, which clip values outside 0–1.

Ideas to keep: a tapered FFT, log band mapping, weighted smoothing across neighbouring bands, peak decay, temporal averaging, and one analysis that feeds several styled views.
Ideas to drop: X11 code, DSP tied to frame rate, the `#request` preprocessor, and one audio capture per instance.

### vibe

[TornaxO7/vibe](https://github.com/TornaxO7/vibe) is the closest existing project and the baseline to beat.
It uses Rust, wgpu 29, and smithay-client-toolkit 0.20, has about 15k lines, and is AGPLv3; its last commit was `7b17437` on 2026-08-25.
These gaps are confirmed in its source:

- It offers only the background and overlay layers, and click-through works only on overlay.
- Hot reload works only in its preview window, not on the desktop.
- Audio goes through cpal input streams, and users route the sink monitor by hand.
- The FFT runs once per rendered frame, so smoothing depends on frame rate.
- Surfaces cover the whole output, and sizes use logical pixels.
- It aborts when premultiplied alpha is unavailable.

Run vibe once on the target machine as a baseline for transparency and feel.

### Other prior art

| Project              | Stack               | Use here                                |
| -------------------- | ------------------- | --------------------------------------- |
| cava                 | C                   | Reference DSP in `cavacore.c`           |
| audioMotion-analyzer | JavaScript          | Reference for band scales and weighting |
| WayVes               | C++, GTK4, PipeWire | GLSL hot reload on layer-shell          |
| hyprglaze            | Zig, OpenGL         | Shadertoy-style uniforms; Hyprland only |
| cava-bg              | Rust, wgpu          | Layer-shell bars fed by cava            |
| shaderbg             | Rust, OpenGL        | Shadertoy import; no audio              |
| projectM             | C++                 | MilkDrop presets; audio feature ideas   |

Read upstream source with the dotfiles `src` command, for example `src get github.com/karlstav/cava`.

## Design

### Pipeline

```text
PipeWire sink monitor (48 kHz, stereo, f32)
  → capture callback → sample ring buffer
  → analysis thread, hop 5–10 ms → timestamped band frames
  → render loop, once per frame callback → motion, per-sink delay
  → views → one layer surface per view per output
```

Analysis runs on audio time, and rendering runs on display time.
Each frame, the render loop takes the newest analysis frame that is older than the sink delay and advances motion by the real `dt`.

### Modules

Start with one crate and these modules; split crates only when a boundary proves itself.

| Module    | Owns                                        |
| --------- | ------------------------------------------- |
| `audio`   | PipeWire stream, sink tracking, sample ring |
| `dsp`     | FFTs, bands, decibels, tilt, gain, features |
| `wayland` | Outputs, layer surfaces, frame callbacks    |
| `render`  | wgpu device, pipelines, uniforms, textures  |
| `config`  | TOML loading, validation, hot reload        |
| `ctl`     | Control socket: hide, show, reload, set     |

### Audio capture

- Use the `pipewire` crate with `stream.capture.sink=true` on an input stream to capture the default sink's monitor.
- Use `target.object` to pin a sink; `node.target` is deprecated.
- Track the default sink so the per-sink delay follows device changes.
- cpal 0.18 has a native PipeWire host, but nobody has checked whether it can capture a sink monitor; prefer the `pipewire` crate.

### Analysis

These are proposed defaults; tune them by eye on real music.

- Use 48 kHz stereo and Hann windows.
- Use several FFT sizes: 8192 for bass (5.9 Hz bins), 2048–4096 for mids, and 512 for treble, split near 150 Hz and 1.5 kHz.
  - cava uses two sizes at 48 kHz (8192 and 4096) and reads the large one below 100 Hz.
- Use a hop of 256–512 samples (5–10 ms), which gives 75% or more overlap.
- Use log-spaced bands of 1/6 to 1/3 octave from about 30 Hz to 16 kHz.
  - Take the maximum bin in each band and interpolate band edges on a log scale, as audioMotion does.
  - Give every band at least one bin; cava pushes low edges up to do this.
- Map decibels over a fixed window of about −85 to −25 dB to 0–1.
- Tilt the spectrum up by 3–4.5 dB per octave so typical music looks level.
- Add automatic gain like cava's `autosens`: drop fast on overshoot (about 0.6 s to halve) and rise slowly when audio plays (about 11 s to double).

### Motion

- Express every constant in milliseconds and scale it by the real `dt`, never by frame count.
- Use an asymmetric envelope: attack in 5–15 ms and release in 150–400 ms with an accelerating fall.
- Add peak caps that hold for 500 ms and then fall over about 750 ms, as audioMotion does.
- Allow optional smoothing across neighbouring bands, like GLava's weighted kernel.
- Stop requesting frames when audio is silent and every envelope has settled; resume on new audio.

cava's fall is `out = peak · (1 − fall² · g)` with `fall += 0.028` per frame and `g = (66/fps)^2.5 · 2/0.77`.
Port it to time units before using it.

### Views and rendering

A view is one shader plus a placement and its parameters, and all views share one analysis.

- A placement is an edge (left, right, top, bottom) with a thickness and margins, or a point with a size, on chosen outputs.
- Size each surface to its view instead of covering the whole output, because a transparent 4K surface costs fill work every frame.
- Use physical pixels and handle output scale changes.
- Pick `PreMultiplied` or `PostMultiplied` alpha from the surface capabilities and write matching pixels; `Auto` does not promise transparency.
- Use the `Fifo` present mode; use `Mailbox` only when the capabilities list it.
- Draw at most once per frame callback.

### Shader interface (draft v0)

otorama adds a preamble with these inputs to each shader:

| Input        | Content                                      |
| ------------ | -------------------------------------------- |
| `time`, `dt` | Clock in seconds                             |
| `resolution` | Surface size in physical pixels, plus scale  |
| `bands`      | Smoothed band values per channel, 0–1        |
| `peaks`      | Peak cap positions per channel               |
| `history`    | Texture of bands over recent frames          |
| `features`   | Band energies, onset, loudness, stereo width |
| `previous`   | This view's last frame, for trails and glow  |
| `params`     | User values from the view's config           |

The features come from these sources:

- **Band energies** use audioMotion's ranges: bass 20–250 Hz, low-mid 250–500 Hz, mid 500 Hz–2 kHz, high-mid 2–4 kHz, treble 4–16 kHz.
- **Fast and slow averages** of each energy give a hit signal when divided, like MilkDrop's `bass` and `bass_att`; the MilkDrop constants are unverified.
- **Onset** uses half-wave rectified spectral flux on the log bands with an adaptive threshold; SuperFlux is a later refinement.
- **Loudness** is momentary LUFS over a 400 ms window, available from the `ebur128` crate.
- **Stereo width** uses the L/R correlation and the mid/side ratio.

Ship built-in edge bars and a radial ring so the GLava setup can move over first.

### Config, hot reload, and control

- Use TOML; each view names its shader file, placement, and parameters.
- Watch config and shader files with `notify`.
- Compile new pipelines off the render path, swap them in only on success, and keep the old pipeline when compilation fails.
- Print naga's WGSL errors with their source spans.
- Add a control socket (`otorama ctl`) to hide, show, reload, and set parameters live.

### Wayland surfaces

- Default to `Layer::Bottom`, which sits above the wallpaper and below windows; allow a per-view override.
- Anchor each surface to its view's edge and set the exclusive zone to −1 so windows ignore it.
- Set an empty `wl_region` as the input region for click-through; `None` makes the whole surface take input.
- Set `KeyboardInteractivity::None`.
- Commit first with no buffer, and render only after the configure event.
- Use the layer namespace `otorama` so compositors can match rules.
  - Hyprland's Lua config uses `hl.layer_rule` with `match.namespace` and the keys `blur`, `ignore_alpha`, and `no_anim`.
- Frame callbacks are hints and may stop while a surface is hidden; do not use them as a clock.

### A/V sync

The capture stream receives audio before the listener hears it, earlier by the sink's output delay.
HDMI TVs and Bluetooth often add 100–300 ms; nobody has measured the target devices yet.
GLava's lag of about 135 ms may hide this delay today, so a low-latency otorama without compensation could look worse on Bluetooth.

- Delay the visuals by a per-sink offset in milliseconds, keyed by sink node name and switched when the default sink changes.
- PipeWire reports graph and device latency (`SPA_PARAM_Latency`, and `pw_time.delay` through `Stream::time()`), but not the full acoustic delay of a TV or Bluetooth device.
- `Stream::time()` on the capture stream describes that stream's path, not the remaining playback delay.
- Calibrate each sink with a click track and a slow-motion phone video.

## Stack

Versions are the latest stable releases on 2026-10-11.

| Need            | Crate                    | Version |
| --------------- | ------------------------ | ------- |
| Layer surfaces  | `smithay-client-toolkit` | 0.21.1  |
| Event loop      | `calloop`                | 0.14.5  |
| GPU             | `wgpu`                   | 30.0.1  |
| Shader compiler | `naga`                   | 30.0.1  |
| Shader imports  | `naga_oil`               | 0.23.0  |
| Window handles  | `raw-window-handle`      | 0.6.2   |
| Audio           | `pipewire`               | 0.10.1  |
| FFT             | `realfft`                | 3.5.0   |
| Loudness        | `ebur128`                | 0.1.10  |
| File watching   | `notify`                 | 8.2.0   |
| Config          | `serde`, `toml`          | 1.x     |

Connect wgpu to a layer surface with `Instance::create_surface_unsafe(SurfaceTargetUnsafe::RawHandle)` and Wayland display and window handles, and keep both alive.
The SCTK wgpu example still targets wgpu 0.19, so treat it as a sketch of the connection only.
The `pipewire` crate's docs.rs build failed; use the [pipewire-rs docs](https://pipewire.pages.freedesktop.org/pipewire-rs/).
naga's GLSL frontend accepts Vulkan GLSL 450/460, not Shadertoy code as-is.

## Target machine

- Arch Linux with Hyprland 0.56.x, configured in Lua (`hl.*`).
- AMD Strix Halo with an RDNA 3.5 iGPU on Mesa (RADV).
- One LG TV over HDMI at 3840×2160, 60 Hz (120 Hz capable), scale 1.
- PipeWire 1.6.9 with pipewire-pulse; audio goes to the TV or to Bluetooth headphones.
- An mpvpaper video wallpaper on the background layer.
- Window gaps leave screen margins of top 85, right 90, bottom 27, and left 124 pixels; the eww bar uses the top.

## GLava setup to replace

The author's GLava config is in `~/dotfiles/config/glava/`, and the first real milestone reproduces it.

| Instance    | Geometry (x y w h) | Look                                      |
| ----------- | ------------------ | ----------------------------------------- |
| `bars_rc`   | 1 195 120 2110     | Left edge; 2 px bars, 2 px gaps, gain 356 |
| `bars_r_rc` | 3740 316 100 1924  | Right edge; channels swapped, gain 200    |
| `radial_rc` | −200 −200 400 400  | Ring; 80 bars, radius 90, width 3.5       |

- Edge bars grow inward from the screen edge, with bass at both ends and treble in the middle, one channel per half.
- Both bar sets taper the lowest and highest bands.
- The color is `#6380ec`, brighter as a bar grows.
- The right bars start lower to clear the taller workspace bar.

## Risks

- **Transparency:** RADV on Hyprland may not offer a non-opaque alpha mode to wgpu; the spike settles this, and so does vibe running transparent.
- **Latency:** sink delay may not be readable, so plan on manual per-sink offsets.
- **Feel:** if the author prefers GLava's laggy look side by side, the gains shift to correctness and features; the motion constants stay tunable.
- **Hot reload:** pipeline compile time has no documented bound, so compilation must never block a frame.
- **Frame callbacks:** they may stop while a surface is hidden, so wake rendering from audio as well.

## Open decisions

1. **Default layer:** bottom (behind windows, visible in the gap margins) or overlay (above windows).
   - The recommendation is bottom; first check where GLava draws today under XWayland.
2. **Shader language:** WGSL (recommended, native to naga, clear errors) or GLSL with a Shadertoy adapter.
3. **Working style:** the author wants to write much of this himself as his first project of this kind; agree whether agents pair and explain or build whole pieces.
4. **Config schema:** the exact TOML shape and shader inputs beyond the draft above.

## Roadmap

1. **Spike:** one layer surface, PipeWire monitor capture, one FFT, and one WGSL bar shader; it may be throwaway.
   - It answers transparency, click-through, and latency on the target; run vibe once beside it.
2. **Core:** the analysis thread, DSP v1 (bands, decibels, tilt, gain), time-based motion, and a render loop that idles.
3. **Parity:** built-in edge bars and radial ring that match the GLava setup, then the swap in dotfiles.
4. **Editing:** TOML and WGSL hot reload with error reports, and the control socket.
5. **Sync:** the per-sink delay table and calibration steps.
6. **Shaders:** history, features, the feedback texture, and example shaders.
7. **Beta:** a README, examples, a PKGBUILD, and the crates.io and AUR releases.

## Dotfiles changes at parity

- `cmds/internal/hyprd/session/init.go`: start otorama instead of the `glavaExecs` list, or use a user unit in `config/systemd/user/`.
- `cmds/internal/hyprd/session/share.go`: replace `killall glava` on enter with `otorama ctl hide`, and the `dispatchGLava` restart on exit with `otorama ctl show`.
- `config/hypr/hyprland.lua`: remove the GLava window rule, and add an `hl.layer_rule` for namespace `otorama` if needed.
- `config/otorama/`: add the config and shaders, linked by home setup.
- `packages/aur.lst`: replace `glava` with `otorama`, or add a local `packages/otorama/PKGBUILD`.
- `README.md` and `cmds/cmd/hyprd/README.md`: replace the GLava mentions.
- `config/glava/`: delete it after the swap.

A separate Quickshell lock-screen visualizer (`config/quickshell/lock/Glava.qml`) was in progress in dotfiles on 2026-10-11; it is separate work.

## Release

- **crates.io:** create a token with the `publish-new` scope limited to `otorama`, then run `cargo login`.
  - crates.io removes names held for a long time without real development, so publish at beta.
  - An owner can delete a crate only within 72 hours of publishing, or under strict download limits.
- **AUR:** create an SSH key at `~/.ssh/aur`, add a `Host aur.archlinux.org` block with `User aur`, and add the public key to the AUR account.
  - Publish `otorama` for tagged releases and `otorama-git` for the branch.
  - The AUR deletes packages that install nothing useful, so publish at beta.
- Package `UNLICENSE` and set `license=('Unlicense')` in the PKGBUILD.

## References

- [GLava](https://github.com/jarcode-foss/glava)
- [vibe](https://github.com/TornaxO7/vibe) and [shady](https://github.com/TornaxO7/shady)
- [cava](https://github.com/karlstav/cava) and [audioMotion-analyzer](https://github.com/hvianna/audioMotion-analyzer)
- [WayVes](https://github.com/Roonil/WayVes), [hyprglaze](https://github.com/slastra/hyprglaze), [cava-bg](https://github.com/leriart/cava-bg), [shaderbg](https://github.com/hbatagelo/shaderbg)
- [wlr-layer-shell protocol](https://wayland.app/protocols/wlr-layer-shell-unstable-v1)
- [SCTK examples](https://github.com/smithay/client-toolkit/tree/master/examples)
- [PipeWire latency model](https://docs.pipewire.org/page_latency.html)
- [Hyprland layer rules](https://github.com/hyprwm/hyprland-wiki/blob/main/content/configuring/core/rules/layer-rules.md)
