# Prior art: cloud-game, Selkies, and what they teach about seams and the DS

Research for [#4](https://github.com/UnstoppableMango/mangomu/issues/4), under the map [#1](https://github.com/UnstoppableMango/mangomu/issues/1).
The question is what mangomu can learn from [giongto35/cloud-game](https://github.com/giongto35/cloud-game) (CloudRetro) and similar projects without adopting them.
This is not a case for forking either one.

## Sources and how they were read

Everything below was read from primary sources: the projects' own code, design docs and wikis, and commit history.

| Source | Revision read |
| --- | --- |
| `giongto35/cloud-game` source, `DESIGNv2.md`, `README.md` | [`1fab9ef`](https://github.com/giongto35/cloud-game/tree/1fab9ef07a02cab0e0cdea169e95b5ec6e2e04ed) (2026-09-05), full history of 1294 commits |
| `giongto35/cloud-game` wiki (`FAQ.md`, `Streaming.md`, `WebRTC.md`) | wiki `master` as of 2026-10-22 |
| `selkies-project/selkies` source and `docs/` | [`1b9f739`](https://github.com/selkies-project/selkies/tree/1b9f7395db1c3ea48655a94134b4b391f70e782c) (2026-10-22) |
| `libretro.h` (as vendored by cloud-game) | same cloud-game revision |

Links below are permalinks into those revisions, abbreviated as `cg:` for cloud-game and `selkies:` for Selkies.

**Gaps.**
The cloud-game issue tracker, its wiki FAQ page on the web, and the webrtchacks write-up linked from its README could not be fetched in this session (the requests were blocked).
Issue-tracker evidence is therefore replaced with commit history, which records the same fixes but not the user reports behind them.
DS-specific behaviour of the melonDS and DeSmuME libretro cores is out of scope here and belongs to the core-selection research; this note only covers what the libretro API itself requires of a frontend.

## 1. The host–core boundary

### What cloud-game does

There are two layers.

- **`app.App`** ([cg:pkg/worker/caged/app/app.go](https://github.com/giongto35/cloud-game/blob/1fab9ef07a02cab0e0cdea169e95b5ec6e2e04ed/pkg/worker/caged/app/app.go)) is the interface the rest of the worker sees.
  It is callback-shaped (`SetAudioCb`, `SetVideoCb`, `SetDataCb`), takes input as `Input(port int, device byte, data []byte)`, and emits a single `RawFrame{Data, Stride, W, H}` per video callback.
  It also carries presentation hints the browser needs: `AspectRatio`, `Rotation`, `Flipped`, `Scale`, `PixFormat`, `ViewportSize`.
- **`libretro.Frontend` / `nanoarch`** ([cg:frontend.go](https://github.com/giongto35/cloud-game/blob/1fab9ef07a02cab0e0cdea169e95b5ec6e2e04ed/pkg/worker/caged/libretro/frontend.go), [cg:nanoarch.go](https://github.com/giongto35/cloud-game/blob/1fab9ef07a02cab0e0cdea169e95b5ec6e2e04ed/pkg/worker/caged/libretro/nanoarch/nanoarch.go)) is a cgo libretro frontend that `dlopen`s a core and implements the callbacks.

`caged.Manager` is nominally a registry of "app" modules, but only `libretro` exists, and it is loaded by reflecting over the config struct for fields named `Emulator` and `Recording` ([cg:caged.go](https://github.com/giongto35/cloud-game/blob/1fab9ef07a02cab0e0cdea169e95b5ec6e2e04ed/pkg/worker/caged/caged.go)).
The `Caged` wrapper then exposes libretro-specific operations (`Load(games.GameMetadata, path)`, `EnableCloudStorage`, `SetSaveOnClose`, `ReloadFrontend`) that the worker calls by concrete type, so the `App` interface is not the whole seam in practice.

### How faithful the libretro wrapping is

It is a partial frontend, tuned to the cores cloud-game ships.

- **Process-global singleton.** C callbacks reach Go through one global, `Nan0`, guarded by a one-slot channel with the comment "this thing forbids concurrent use of the emulator" and "!to remove when isolated properly" ([cg:nanoarch.go#L130-L144](https://github.com/giongto35/cloud-game/blob/1fab9ef07a02cab0e0cdea169e95b5ec6e2e04ed/pkg/worker/caged/libretro/nanoarch/nanoarch.go#L130-L144)).
  One worker process can run one core, ever.
  The wiki FAQ admits the consequence: "game sessions are not fully separated, so the game session can lag over time" ([wiki FAQ](https://github.com/giongto35/cloud-game/wiki/FAQ)).
- **Input devices.** `input_state` answers `RETRO_DEVICE_JOYPAD`, `ANALOG`, `KEYBOARD` and `MOUSE` only ([cg:nanoarch.c#L202-L264](https://github.com/giongto35/cloud-game/blob/1fab9ef07a02cab0e0cdea169e95b5ec6e2e04ed/pkg/worker/caged/libretro/nanoarch/nanoarch.c#L202-L264)).
  There is no `RETRO_DEVICE_POINTER`, which is the device libretro DS cores read the touchscreen from.
- **Environment calls.** The handled set is av info, geometry, rotation, pixel format, directories, variables, HW render, controller info, keyboard callback, audio buffer status, minimum audio latency and frame time callback ([cg:nanoarch.go#L779-L910](https://github.com/giongto35/cloud-game/blob/1fab9ef07a02cab0e0cdea169e95b5ec6e2e04ed/pkg/worker/caged/libretro/nanoarch/nanoarch.go#L779-L910)).
  `RETRO_ENVIRONMENT_GET_MICROPHONE_INTERFACE` is not handled, so mic-using cores see no microphone.
  Core options are plain YAML maps per core with per-ROM overrides, and quirks become named "hacks" (`skip_hw_context_destroy`, `skip_same_thread_save`) in config ([cg:config.yaml#L149-L351](https://github.com/giongto35/cloud-game/blob/1fab9ef07a02cab0e0cdea169e95b5ec6e2e04ed/pkg/config/config.yaml#L149-L351)).
- **Persistence.** SRAM is written only inside `Save()`, right after a full save state, and that only runs on an explicit save, on autosave, or on quit when a save already exists ([cg:frontend.go#L295-L301](https://github.com/giongto35/cloud-game/blob/1fab9ef07a02cab0e0cdea169e95b5ec6e2e04ed/pkg/worker/caged/libretro/frontend.go#L295-L301), [#L482-L500](https://github.com/giongto35/cloud-game/blob/1fab9ef07a02cab0e0cdea169e95b5ec6e2e04ed/pkg/worker/caged/libretro/frontend.go#L482-L500)).
  An in-game save made in a session that never pressed "save" is lost on a crash.
- **Frame pacing.** The core is driven by the host's own loop: sleep until 1 ms before the deadline, then busy-spin on a nanosecond clock; a late frame sets `skipVideo` ([cg:frontend.go#L321-L380](https://github.com/giongto35/cloud-game/blob/1fab9ef07a02cab0e0cdea169e95b5ec6e2e04ed/pkg/worker/caged/libretro/frontend.go#L321-L380)).
  Pacing therefore lives in the libretro frontend, not above the core seam.

## 2. The browser protocol

### Signalling

Signalling is a custom JSON-over-WebSocket protocol, `{id?, t, p?}` with numeric packet types ([cg:pkg/api/api.go](https://github.com/giongto35/cloud-game/blob/1fab9ef07a02cab0e0cdea169e95b5ec6e2e04ed/pkg/api/api.go)).
The browser talks only to the coordinator; the coordinator relays SDP and ICE to the worker over a second WebSocket, and the worker makes the offer ([wiki Streaming.md](https://github.com/giongto35/cloud-game/wiki/Streaming)).
Game control (start, quit, save, load, reset, change player) travels over that same signalling socket, not the DataChannel, and the start response carries the video geometry (`AppVideoInfo{w,h,a,s,flip,rot}`) ([cg:pkg/api/worker.go](https://github.com/giongto35/cloud-game/blob/1fab9ef07a02cab0e0cdea169e95b5ec6e2e04ed/pkg/api/worker.go)).

### Input over the DataChannel

- The default channel is unordered with `MaxRetransmits: 0`, so input is fire-and-forget ([cg:webrtc.go#L30-L36](https://github.com/giongto35/cloud-game/blob/1fab9ef07a02cab0e0cdea169e95b5ec6e2e04ed/pkg/network/webrtc/webrtc.go#L30-L36)).
- Gamepad input is a full-state snapshot: a 16-bit button mask plus four int16 axes, 10 bytes, little-endian ([cg:retropad.js#L4-L29](https://github.com/giongto35/cloud-game/blob/1fab9ef07a02cab0e0cdea169e95b5ec6e2e04ed/web/js/input/retropad.js#L4-L29), [cg:input.go](https://github.com/giongto35/cloud-game/blob/1fab9ef07a02cab0e0cdea169e95b5ec6e2e04ed/pkg/worker/caged/libretro/nanoarch/input.go)).
  A snapshot makes a lost packet harmless only if another one follows, but the client sends only on change, polled from `requestAnimationFrame`.
  A lost release packet on an unreliable channel leaves a button held until the next change.
- Keyboard and mouse get their own channels, opened only when the core config says `kbMouseSupport`, and those use big-endian framing while the gamepad uses little-endian ([cg:coordinatorhandlers.go#L198-L205](https://github.com/giongto35/cloud-game/blob/1fab9ef07a02cab0e0cdea169e95b5ec6e2e04ed/pkg/worker/coordinatorhandlers.go#L198-L205)).
- The device type is implied by which channel a message arrived on, so the message format does not describe itself.
- The mouse sends relative deltas scaled against a hard-coded 640×480 "game resolution" ([cg:pointer.js#L10-L26](https://github.com/giongto35/cloud-game/blob/1fab9ef07a02cab0e0cdea169e95b5ec6e2e04ed/web/js/input/pointer.js#L10-L26)).
  Nothing sends an absolute pointer position.
- `touch.js` is an on-screen virtual gamepad that produces RetroPad bits, not touches on the game picture.

### Transport-agnostic?

No.
Input handling is bound to `room.WithWebRTC(user.Session).OnMessage(...)`, and the frontend's stream code assumes a WebRTC `<video>` element.

### Selkies as the contrast

Selkies streams over WebSockets by default and offers WebRTC as an opt-in transport (`--mode=webrtc`) ([selkies:docs/design.md#L34](https://github.com/selkies-project/selkies/blob/1b9f7395db1c3ea48655a94134b4b391f70e782c/docs/design.md#L34)).
"Both transports (WebSockets and WebRTC) feed client messages into one WebRTCInput handler, so input authority and fallback behavior stay identical across modes" ([selkies:input_handler.py](https://github.com/selkies-project/selkies/blob/1b9f7395db1c3ea48655a94134b4b391f70e782c/src/selkies/input_handler.py)).
Its input messages are self-describing comma-separated text (`kd,<keysym>`, `ku,…`, `js,<index>,…`, `REQUEST_KEYFRAME`), so the same parser works whichever pipe delivered them.
Its WebSocket mode needs only one TCP port and uses WebCodecs on the client, while WebRTC mode brings STUN/TURN and UDP ([selkies:docs/design.md#L48-L50](https://github.com/selkies-project/selkies/blob/1b9f7395db1c3ea48655a94134b4b391f70e782c/docs/design.md#L48-L50), [selkies:docs/faq.md#L11](https://github.com/selkies-project/selkies/blob/1b9f7395db1c3ea48655a94134b4b391f70e782c/docs/faq.md#L11)).

## 3. Sessions, coordinator and workers, and pods

- The coordinator is the web frontend, the signalling relay and the load balancer.
  Workers dial in over WebSocket and register; the user pings a list of workers and the coordinator picks the fastest ([cg:DESIGNv2.md](https://github.com/giongto35/cloud-game/blob/1fab9ef07a02cab0e0cdea169e95b5ec6e2e04ed/DESIGNv2.md)).
- **A worker runs one game.**
  "Workers support only one game at a time" ([cg:coordinator/worker.go#L140-L143](https://github.com/giongto35/cloud-game/blob/1fab9ef07a02cab0e0cdea169e95b5ec6e2e04ed/pkg/coordinator/worker.go#L140-L143)); the wiki says "workers have 1:1 mapping to rooms or games".
  The worker's `Router` holds exactly one `Room`, and the room closes when its last user leaves ([cg:room.go#L141-L223](https://github.com/giongto35/cloud-game/blob/1fab9ef07a02cab0e0cdea169e95b5ec6e2e04ed/pkg/worker/room/room.go#L141-L223)).
  This was forced by the `Nan0` singleton rather than chosen, but it means cloud-game's worker is already shaped like pod-per-session; what it lacks is anything that creates workers on demand.
  Scaling is "add more workers", deployed as long-lived docker-compose hosts ([cg:.github/workflows/cd](https://github.com/giongto35/cloud-game/tree/1fab9ef07a02cab0e0cdea169e95b5ec6e2e04ed/.github/workflows/cd)).
- A room is many users to one app: media is encoded once and fanned out to every peer, and each peer's input goes to the port it chose (`GameSession.Index`).
- Sessions survive a worker change only through save states in S3-compatible storage keyed by room id ([cg:config.yaml#L416-L424](https://github.com/giongto35/cloud-game/blob/1fab9ef07a02cab0e0cdea169e95b5ec6e2e04ed/pkg/config/config.yaml#L416-L424)).
- Slot reservation has had races fixed repeatedly (`ddfc924` "Fix some user slot race conditions", `f09500f`, `5b4f74e`); the coordinator's user handlers now carry a long comment explaining when a busy worker may be reused.
- In a container the worker needs `CLOUD_GAME_WEBRTC_SINGLEPORT` (one UDP port for all ICE) and sometimes `iceIpMap` to advertise the host IP instead of the bridge IP ([cg:docker-compose.yml](https://github.com/giongto35/cloud-game/blob/1fab9ef07a02cab0e0cdea169e95b5ec6e2e04ed/docker-compose.yml), [cg:config.yaml#L426-L460](https://github.com/giongto35/cloud-game/blob/1fab9ef07a02cab0e0cdea169e95b5ec6e2e04ed/pkg/config/config.yaml#L426-L460)).
  OpenGL cores need an X server; the compose file runs an Xvfb sidecar.

## 4. The DS: cores, dual screens, touch and mic

- **cloud-game has never supported the DS.**
  A `desmume_libretro.so` binary was committed in the 2019 "Integrate with RetroArch" change (`c7caa87`) and deleted in the 2020 core-manager change (`1fcf34e`), but no core map at any revision routes a DS ROM to it.
  The current core list is gba, pcsx, mame, nes, snes, n64 and dos ([cg:config.yaml#L271-L351](https://github.com/giongto35/cloud-game/blob/1fab9ef07a02cab0e0cdea169e95b5ec6e2e04ed/pkg/config/config.yaml#L285-L351)).
- **Touch would not work** even if a DS core loaded, because there is no `RETRO_DEVICE_POINTER` and the client sends no absolute coordinates.
- **Mic would not work**, because the microphone environment call is unanswered and there is no audio uplink from the browser.
- **Dual screens.** cloud-game's video path is one frame of one size per tick, sent as one track.
  Libretro gives the same constraint from the other side.
  `RETRO_DEVICE_POINTER` reports coordinates in `[-0x7fff, 0x7fff]` across "the area that is passed to the frontend", meaning the whole frame the core output ([libretro.h#L268-L300](https://github.com/giongto35/cloud-game/blob/1fab9ef07a02cab0e0cdea169e95b5ec6e2e04ed/pkg/worker/caged/libretro/nanoarch/libretro.h#L268-L300)).
  A libretro DS core therefore composes both screens into one frame according to its own layout option, and a touch has to be mapped into that composed frame, not into the bottom screen alone.
  Any frontend that shows the screens apart has to know the core's layout to translate a touch back.
- **Microphone API.** `RETRO_ENVIRONMENT_GET_MICROPHONE_INTERFACE` is still marked `EXPERIMENTAL` in libretro.h ([#L2471](https://github.com/giongto35/cloud-game/blob/1fab9ef07a02cab0e0cdea169e95b5ec6e2e04ed/pkg/worker/caged/libretro/nanoarch/libretro.h#L2471)).
- **Secure context.** Selkies' docs note that outside `localhost`, plain HTTP is not a secure context, and "the clipboard, gamepads, pointer lock, and the microphone and webcam need one" ([selkies:docs/native.md#L100](https://github.com/selkies-project/selkies/blob/1b9f7395db1c3ea48655a94134b4b391f70e782c/docs/native.md#L100)).
  For mangomu, served over LAN or Tailscale with no auth, that means the mic (and the Gamepad API) needs HTTPS.

## 5. Latency pitfalls and fixes, from history

- **Drop, never queue.** Every GStreamer stage is `queue max-size-buffers=1` and `appsink max-buffers=1 drop=true sync=false`, and the Go side does a non-blocking send into the encoder, dropping the frame if it is busy ([cg:gstreamer.go#L398-L401](https://github.com/giongto35/cloud-game/blob/1fab9ef07a02cab0e0cdea169e95b5ec6e2e04ed/pkg/worker/media/gstreamer.go#L398-L401), [#L449-L486](https://github.com/giongto35/cloud-game/blob/1fab9ef07a02cab0e0cdea169e95b5ec6e2e04ed/pkg/worker/media/gstreamer.go#L449-L486)).
  `f2cb7eb` (2026-07) raised that channel from 1 to 3 "in case if an encoder slower than 16ms": a one-slot buffer dropped too much when the encoder was slow.
- **Encoder settings.** Defaults are VP8 realtime: `deadline=1 cpu-used=6 end-usage=cbr`, 3.2 Mbps, error-resilient, keyframe interval 300 (the VP9 profile adds `lag-in-frames=0`).
  Opus uses `restricted-lowdelay` with 5 ms frames ([cg:config.yaml#L353-L389](https://github.com/giongto35/cloud-game/blob/1fab9ef07a02cab0e0cdea169e95b5ec6e2e04ed/pkg/config/config.yaml#L353-L397)).
  H.264 (x264 `tune=zerolatency`) is optional because it is GPL.
- **Encoder seam.** In June 2026 (`728b2a4`, #504) cloud-game threw away about 3,200 lines of hand-written cgo glue to x264, libvpx and libyuv and replaced it with a GStreamer pipeline built from a config string.
  The reason given was being able to "change encoders to any supported by GStreamer".
- **Rebuild, don't renegotiate.** When the core changes geometry, the pipeline is torn down and rebuilt "because changing the pipeline parts proves to be unreliable", with stale frames dropped ([cg:gstreamer.go#L404-L430](https://github.com/giongto35/cloud-game/blob/1fab9ef07a02cab0e0cdea169e95b5ec6e2e04ed/pkg/worker/media/gstreamer.go#L404-L430)).
- **Client-side transforms.** Vertical flip and rotation moved to a CSS transform on the `<video>` (`e3b16d5`, [cg:stream.js](https://github.com/giongto35/cloud-game/blob/1fab9ef07a02cab0e0cdea169e95b5ec6e2e04ed/web/js/stream.js)).
- **VFR is a trap for CFR cores.** "VFR for CFR cores leads to noticeable video stutter"; only N64 enables it ([cg:config.yaml](https://github.com/giongto35/cloud-game/blob/1fab9ef07a02cab0e0cdea169e95b5ec6e2e04ed/pkg/config/config.yaml#L251-L257)).
- **Audio drift.** The audio-buffer-status callback is fed an occupancy computed from the drift between wall-clock and expected frame time ([cg:nanoarch.go#L712-L740](https://github.com/giongto35/cloud-game/blob/1fab9ef07a02cab0e0cdea169e95b5ec6e2e04ed/pkg/worker/caged/libretro/nanoarch/nanoarch.go#L712-L740)).
- **Network hygiene.** Smaller fixes in the history: an ICE restart on the client with candidate buffering reset (`7442ff7`), audio sent unordered and unreliable since 2019 (`517df7d`), and Pion's default interceptors made switchable for performance (`aa10008`, `webrtc.disableDefaultInterceptors`).
- **Measure on screen.** Selkies shows a "Poor connection" mark from frame-loss thresholds and puts encoder, decoder and path rows (Host UDP / Relay / TCP) at the top of its stats ([selkies:docs/faq.md#L44-L70](https://github.com/selkies-project/selkies/blob/1b9f7395db1c3ea48655a94134b4b391f70e782c/docs/faq.md#L44-L70)).
  It also collects the detailed stats only while they are on screen, so that measuring does not perturb latency.

## Lessons for mangomu

1. **Put the core seam above libretro, not at it.**
   cloud-game's `App` is close to the right shape (callbacks out, input in, presentation hints), but libretro concepts leak through it (`PixFormat` as a libretro enum, `device byte`, RetroPad bitmask payloads), and the worker still downcasts to the concrete type.
   A home-grown core should implement mangomu's interface, with libretro as one adapter behind it.
2. **Make the core seam multi-surface and multi-input from day one.**
   It needs N video surfaces (top and bottom screen, each with size and pixel format), absolute pointer input in a surface's own coordinates, and an audio-in path.
   cloud-game's single `RawFrame` and joypad/keyboard/mouse set cannot express a DS, and adding them later would touch every layer.
   The libretro adapter's job is to compose and decompose: it knows the core's layout and maps a bottom-screen touch into composed-frame pointer coordinates.
3. **Make input self-describing and idempotent, and keep the transport out of it.**
   Follow Selkies: one decoder for input messages, fed by whichever pipe carries them.
   Tag each message with device and surface rather than inferring them from the channel.
   Send full state, and on an unreliable channel resend it periodically (or on a short timer while anything is held) so that a lost "release" heals itself.
4. **Separate control from media.**
   cloud-game keeps game control (start, save, reset) on the signalling WebSocket and only input on the DataChannel.
   This is worth keeping: control wants reliability and a reply; input wants neither.
5. **Session = process = pod is the natural grain.**
   cloud-game arrived at one game per worker by accident (a global singleton) and then spent years on slot-reservation races in a central coordinator.
   For mangomu, model a session as something that owns exactly one core instance, and keep the "find or create the session" decision behind an interface.
   v1 implements it as "the one pod"; pod-per-session later implements it with the Kubernetes API, and there is no slot counting to race.
6. **Persist SRAM independently of save states.**
   Flush on the core's own signal or a short timer and on shutdown, to storage that survives the pod (the map requires it).
   Do not do what cloud-game does and tie SRAM to the user's "save".
7. **Keep the pipeline drop-not-queue, and expect to rebuild it.**
   Use a bounded encoder input with drop-oldest semantics, rebuild the encoder on geometry change, and make the encoder a config-driven seam so that VP8, H.264 and a hardware encoder are swappable.
8. **Pod networking for WebRTC needs a single UDP port and an advertised IP** (cloud-game's `singlePort` and `iceIpMap`), or `hostNetwork`.
   Over Tailscale the advertised candidate must be the tailnet address.
9. **HTTPS is a requirement, not polish**, for the mic and the Gamepad API off `localhost`.

## Anti-patterns to avoid

- A process-global emulator singleton reached from C callbacks (`Nan0`).
- Inferring the input device from the DataChannel label, and using mixed endianness across channels.
- Change-only input sends over an unreliable, unordered channel with no periodic refresh.
- Hard-coded resolution constants in client input scaling (the 640×480 in `pointer.js`).
- Saving SRAM only as a side effect of a save state.
- A config struct reflected over by field name to build a module (`caged.loadLibretro`).
- Shipping a core binary without wiring or testing it (the 2019 `desmume_libretro.so`).
- A central coordinator doing slot accounting for workers it does not own the lifecycle of.

## Open points this raises for other tickets

- **Core seam (#6):** the seam needs multiple video surfaces, absolute pointer input per surface, and audio input, or the DS cannot pass through it.
  The libretro adapter owns layout composition and touch-coordinate translation.
- **Frontend protocol (#8):** the decision should be framed as "one self-describing input schema, with WebRTC DataChannel as v1's carrier", not "the DataChannel protocol".
  Selkies shows that a WebSocket + WebCodecs transport is a real option for a LAN-only, TURN-less deployment.
  Mic capture needs a secure context, so the deployment needs HTTPS (for example a Tailscale certificate).
- **Dual screens (#7):** whether the screens are one composed stream or two tracks, touch has to reach the core in composed-frame coordinates under libretro, so the layout must be known on the host side.
  If the screens are split, the client needs per-surface geometry, which cloud-game sends today as `AppVideoInfo` in the start response.
