# Research: WebRTC media pipeline for a Go host under 100ms

Question ([#3](https://github.com/UnstoppableMango/mangomu/issues/3)): what should carry raw emulator frames and audio from a Go process to a browser over WebRTC, within a 100ms input-to-photon budget on LAN, with software encoding only?

Researched 2026-10-22. Versions current on that date: `pion/webrtc/v4` v4.2.25, `pion/mediadevices` v0.10.1, `pion/interceptor` v0.1.49, `go-gst/go-gst` v1.4.1, GStreamer 1.28.7 (nixpkgs), FFmpeg 9.0.2 (nixpkgs, used only for benchmarking).

## Recommendation

Use **Pion WebRTC v4 for transport, and call libvpx and libopus through a thin cgo wrapper that mangomu owns**.

- **Video:** VP8 from libvpx, using `VPX_DL_REALTIME`, `cpu-used` 6 to 8, `lag_in_frames=0`, CBR and error-resilient mode. Hide the codec behind an interface so H.264 via OpenH264 can be added later.
- **Frames:** Upscale each frame 2x with nearest-neighbour before encoding, to 512x768 with both screens stacked, and send it as one video track.
- **Audio:** Opus from libopus with `OPUS_APPLICATION_RESTRICTED_LOWDELAY` and 10ms frames, at 48kHz stereo.
- **Input:** one unordered DataChannel with no retransmits, on the same `RTCPeerConnection`.
- **Latency:** the sender adds the `playout-delay` RTP header extension with min=0 and max=0, the browser sets `jitterBufferTarget = 0`, and audio and video go in separate MediaStreams.
- **Measurement:** the browser reads `getStats()` and `requestVideoFrameCallback` metadata, and the host logs per-stage timestamps.

Why this one:

- It is the smallest stack that meets every constraint. At DS resolution, encoding is only a small slice of the 100ms budget: 1.3ms per frame at 1x and 3.3ms at 2x, on one core of a 2.1GHz Xeon (see [Benchmarks](#cpu-cost-at-ds-resolution-measured)).
- VP8 is mandatory to implement in every WebRTC browser ([RFC 7742](https://datatracker.ietf.org/doc/html/rfc7742), via [MDN: Codecs used by WebRTC](https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats/WebRTC_codecs)). Firefox does not need its separately downloaded OpenH264 plugin to decode it.
- libvpx, libopus and Pion are all permissively licensed (BSD-3-Clause, BSD-3-Clause, MIT). x264 is GPL-2.0-or-later (nixpkgs `meta.license`), and mangomu is MIT.
- The Nix closure for x264, libvpx and libopus together is **58 MiB**, 36 MiB of which is glibc. The GStreamer set that `webrtcbin` needs is **828 MiB** unslimmed.
- cloud-game, the stated prior art, settled on the same shape: Pion for transport, VP8 by default, and Opus with `restricted-lowdelay` at 5ms frames ([cloud-game `pkg/config/config.yaml`](https://github.com/giongto35/cloud-game/blob/master/pkg/config/config.yaml), `encoder:` section).

## Options compared

| | Pion + cgo encoders | GStreamer `webrtcbin` from Go | Pion transport + GStreamer encode (cloud-game today) |
|---|---|---|---|
| Transport | Pure Go ([pion/webrtc](https://github.com/pion/webrtc)) | C, in gst-plugins-bad ([webrtc plugin docs](https://gstreamer.freedesktop.org/documentation/webrtc/index.html)) | Pure Go |
| Encoders | libvpx, x264, OpenH264, SVT-AV1 and libopus, via cgo and pkg-config | Any GStreamer encoder element | GStreamer `appsrc` → encoder → `appsink` |
| DataChannels | `pc.CreateDataChannel`, with `Ordered`, `MaxRetransmits` and `MaxPacketLifeTime` (`datachannel.go`) | `create-data-channel` action signal taking RTCDataChannelInit-style options | Pion |
| Control over latency knobs | Full: you own the RTP packets and the interceptor chain | Through element properties. The receive-side `latency` defaults to 200ms, but that only matters for receiving | Encoders through GStreamer, transport through Pion |
| Go integration | Native | go-gst bindings (cgo and GLib main loop; the README says the bindings do not track GStreamer versions) | Mixed |
| Nix closure | ~58 MiB (x264, libvpx, libopus) | ~828 MiB (gstreamer, base, good, bad, ugly and libnice), before any slimming | Several hundred MiB (base and good at least) |
| Verdict | **Recommended** | Rejected: it hands transport to C, gives up Go-level control, has the largest image, and gains nothing at 256x384 | Possible later if many codecs or scalers are wanted. Not needed for one DS core |

Other credible options were considered and set aside:

- **`pion/mediadevices` codec packages used as a library.** They work, but their defaults do not fit:
  - `x264/bridge.h` applies preset+`zerolatency` and then forces `x264_param_apply_profile(..., "high")`.
  - Opus is hard-coded to `OPUS_APPLICATION_VOIP` (`opus/opus.go`).
  - There is no intra-refresh switch.

  Importing a single subpackage such as `pkg/codec/vpx` is reasonable, but a ~150-line wrapper of mangomu's own keeps every knob visible. Treat mediadevices as reference code.
- **AV1, through SVT-AV1 or libaom.** Browser support is only Chrome 113+ and Firefox 136+, with no Safari (MDN). It also costs 2 to 4x more CPU than VP8 at the same size (benchmarks below). Not worth it at these resolutions.
- **VP9.** It costs the same as VP8 here, but Safari is not listed as supporting it (MDN). Its gains are in quality per bit, which a LAN does not need.

## Codec support across current browsers

Sources: [MDN: Codecs used by WebRTC](https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats/WebRTC_codecs), [RFC 7742](https://datatracker.ietf.org/doc/html/rfc7742) and [RFC 7874](https://datatracker.ietf.org/doc/html/rfc7874).

| Codec | Status | Chrome / Edge | Firefox | Safari |
|---|---|---|---|---|
| VP8 | Mandatory (RFC 7742) | Yes | Yes | 12.1+ |
| H.264 Constrained Baseline | Mandatory (RFC 7742) | 52+ | Yes. Desktop uses the OpenH264 plugin; Android 73+ uses hardware | Yes, with hardware on Apple devices |
| VP9 | Optional | 48+ | Yes | Not listed |
| AV1 | Optional | 113+ | 136+ | Not listed |
| Opus | Mandatory (RFC 7874) | Yes | Yes | Yes |

Pion's `RegisterDefaultCodecs` offers all of these. For H.264 it offers `profile-level-id` 42001f, 42e01f, 4d001f and 64001f, plus H.265 (`mediaengine.go`).

H.264 Constrained Baseline and VP8 profile 0 are both **4:2:0 only**. This matters for pixel art; see [Resolution and chroma](#resolution-and-chroma).

## Encoder settings for low latency

Primary sources are the encoder headers and source.

**x264** ([`common/base.c`](https://github.com/mirror/x264/blob/master/common/base.c), [`x264.h`](https://github.com/mirror/x264/blob/master/x264.h)):

- `--tune zerolatency` sets `rc.i_lookahead=0`, `i_sync_lookahead=0`, `i_bframe=0`, `b_sliced_threads=1`, `b_vfr_input=0` and `rc.b_mb_tree=0`. There is therefore no frame delay inside the encoder.
- `preset ultrafast` additionally turns off CABAC, deblocking, 8x8 transforms and subpel refinement.
- `b_intra_refresh=1` "use[s] periodic intra refresh instead of IDR frames", which spreads the cost of a keyframe across several frames. `x264_encoder_intra_refresh()` can trigger it on demand.
- Use `profile baseline` so the stream matches 42e01f, and `b_annexb=1` and `b_repeat_headers=1` for RTP. The parameter sets must go in-band, because `sprop-parameter-sets` must not appear in WebRTC SDP (MDN, citing RFC 7742).

**libvpx** (from `pion/mediadevices` `pkg/codec/vpx/vpx.go` and its comments; cloud-game's `vp8enc` params):

- deadline `VPX_DL_REALTIME`.
- `VP8E_SET_CPUUSED` 6 to 8. libvpx defaults to 0, which mediadevices notes "dominates per-frame encode latency" under realtime.
- `g_lag_in_frames=0`, `rc_end_usage=VPX_CBR`, `g_error_resilient=VPX_ERROR_RESILIENT_DEFAULT|PARTITIONS`.
- Leave `kf_max_dist` large and force a keyframe on PLI.
- cloud-game uses `keyframe-mode=disabled cpu-used=6 deadline=1 end-usage=cbr error-resilient=default+partitions static-threshold=100 target-bitrate=3200000`.

**OpenH264:** `pion/mediadevices` links a vendored static `libopenh264-linux-{x64,arm64}.a` by default (`openh264_static.go`, build tag `!dynamic`), or links through pkg-config with `-tags dynamic`. Under Nix, use the pkg-config route.

**SVT-AV1** v4.1.0: low delay needs `pred-struct=1` with **CBR** (`rc=2`). VBR is rejected ("VBR Rate control is currently not supported for LOW_DELAY"), and `rtc=1` selects the real-time tuning. mediadevices sets `SVT_AV1_PRED_LOW_DELAY` and defaults to preset 9.

**libaom:** `usage=realtime`, `lag-in-frames=0`, `cpu-used` up to 10 in the library (FFmpeg caps it at 8).

**Intra refresh vs. keyframe on PLI:** a 2x DS keyframe is small, and LAN loss is rare. Start with infinite GOP plus an IDR or keyframe on PLI, which Pion surfaces as RTCP the host can read. Switch to intra refresh only if measurements show bitrate spikes on keyframes.

## CPU cost at DS resolution (measured)

The method:

- Source: FFmpeg 9.0.2 from nixpkgs (`ffmpeg-full`) encoding 600 frames of `testsrc2` at 256x384 and 60fps. The 2x and 3x sources are nearest-neighbour upscales.
- Encoders run single-threaded (`-threads 1`; SVT-AV1 `lp=1`), with bitrates of 1.5, 3 and 5 Mbps for 1x, 2x and 3x.
- Timings come from `-benchmark` user time divided by frames.
- Hardware: 4-vCPU Intel Xeon @ 2.10GHz (cloud VM).

`testsrc2` is flat-coloured moving shapes, a fair proxy for 2D sprite content but not for 3D DS games. Treat these as order-of-magnitude figures and re-measure with real core output.

| Encoder / settings | 1x 256x384 | 2x 512x768 | 3x 768x1152 |
|---|---|---|---|
| x264 `ultrafast` `zerolatency` baseline | 0.42ms (3% of a core) | 1.18ms (7%) | 2.68ms (16%) |
| x264 `veryfast` `zerolatency` | 1.14ms (7%) | 3.17ms (19%) | 6.74ms (40%) |
| OpenH264 (defaults) | 1.22ms (7%) | 3.13ms (19%) | 6.71ms (40%) |
| **libvpx VP8** realtime, cpu-used 8, lag 0 | **1.28ms (8%)** | **3.32ms (20%)** | 7.74ms (46%) |
| libvpx VP9 realtime, speed 8, row-mt | 1.34ms (8%) | 3.64ms (22%) | 6.54ms (39%) |
| SVT-AV1 preset 12, rtc, low-delay CBR | 1.99ms (12%) | 3.83ms (23%) | 6.86ms (41%) |
| SVT-AV1 preset 10, rtc, low-delay CBR | 2.63ms (16%) | 5.63ms (34%) | 10.51ms (63%) |
| libaom realtime, cpu-used 8 | 3.43ms (21%) | 7.59ms (46%) | 12.35ms (74%) |

"% of a core" is CPU-seconds per second of 60fps video.

The **Opus** encoder (libopus, 48kHz stereo, 96kbps, `-application lowdelay`) cost 1.4% of a core with 10ms frames and 2.0% with 5ms frames.

Every option fits on one core at 2x. Encode time per frame, not throughput, is what counts against latency, and at 2x it is at most ~3.3ms for every non-AV1 option.

## Resolution and chroma

The DS output is 256x192 per screen, which makes 256x384 stacked.

At 1x, 4:2:0 shares one chroma sample between 2x2 source pixels, so saturated one-pixel pixel-art edges bleed colour. A nearest-neighbour 2x upscale before encoding gives every source pixel its own chroma sample. It costs ~2ms more encode per frame and ~2x the bitrate, which is trivial on a LAN.

The browser then scales with CSS `image-rendering: pixelated`. 3x adds cost without improving chroma further.

This is reasoning from the 4:2:0 sampling definition, not something measured here. Compare a 1x and a 2x capture by eye once a core runs.

## Opus audio

- WebRTC requires Opus, and the full 6 to 510 kbps range must be supported (MDN, citing RFC 7874).
- Pion's default Opus fmtp is `minptime=10;useinbandfec=1` at 48kHz, 2 channels.
- Use `OPUS_APPLICATION_RESTRICTED_LOWDELAY`, which cloud-game sets as `audio-type=restricted-lowdelay`. Use 10ms frames, or 5ms as cloud-game does, at 64 to 128kbps stereo.
- The emulator's native audio rate must be resampled to 48kHz in the host.
- Send audio and video as **separate MediaStreams**, meaning different stream IDs on the `TrackLocal`s. MDN notes that when audio and video receivers are synchronized, the larger jitter buffer target governs both. Lip-sync would otherwise let the audio jitter buffer delay video frames, and a game prefers fresh frames to perfect sync.

## Input DataChannels alongside media

- A DataChannel created on the same `RTCPeerConnection` rides the same ICE/DTLS transport over SCTP, so nothing extra has to be negotiated or opened through the firewall.
- Pion exposes `Ordered`, `MaxRetransmits` and `MaxPacketLifeTime` (`datachannel.go`). Unordered with no retransmits maps to SCTP partial reliability.
- Recommended shape:
  - An **unordered, `maxRetransmits: 0`** channel for input state. Each message carries a sequence number and the *full* current button state plus touch position, so a lost or late packet is superseded by the next one and never head-of-line blocks.
  - Optionally, a second, reliable ordered channel for discrete control messages such as pause, save or session control.
- Mic uplink, if wanted, should be a browser→host Opus **media track** on the same peer connection, not a DataChannel.

## What governs end-to-end latency

Rough LAN budget (estimates, to be replaced by measurements):

| Stage | Typical | Notes |
|---|---|---|
| Browser input event → DataChannel send | <1ms | |
| Network (LAN) | <1ms one way | |
| Wait for the next emulated frame + emulate | 0 to 17ms | The core runs at ~60Hz. Apply input immediately before stepping a frame |
| Colour convert + encode (2x VP8) | ~3 to 5ms | Measured above |
| Packetize + send | <1ms | Do not add Pion's `pacing` interceptor. It is not in `RegisterDefaultInterceptors`, which adds NACK, RTCP reports, stats and TWCC |
| Receiver jitter buffer | 0 to tens of ms | **Controlled by the hints below** |
| Decode | 1 to 3ms | Visible as `totalDecodeTime` in stats |
| Render / compositor / display | 1 to 2 vsyncs (17 to 33ms) | Display and OS dependent, outside our control |

That totals roughly 25 to 60ms, leaving headroom under 100ms **if the jitter buffer is kept near zero**.

### Playout delay hints

- **RTP header extension `http://www.webrtc.org/experiments/rtp-hdrext/playout-delay`** ([libwebrtc doc](https://webrtc.googlesource.com/src/+/main/docs/native-code/rtp-hdrext/playout-delay/README.md)):
  - It carries two 12-bit fields, min and max delay, in 10ms units (0 to 40950ms).
  - min=max=0 signals "render as quickly as possible".
  - The sender includes it only until RTCP shows it arrived.
  - In libwebrtc's receiver ([`modules/video_coding/timing/timing.cc`](https://webrtc.googlesource.com/src/+/HEAD/modules/video_coding/timing/timing.cc)), `UseLowLatencyRendering()` is true when `min_playout_delay` is zero and `max_playout_delay <= 500ms`. In that case `RenderTime()` returns zero, meaning render as soon as possible.
  - Pion ships the wire format as `rtp.PlayoutDelayExtension` ([pion/rtp `playoutdelayextension.go`](https://github.com/pion/rtp/blob/master/playoutdelayextension.go)). Adding it takes `MediaEngine.RegisterHeaderExtension` with that URI, plus a small interceptor, or a `TrackLocalStaticRTP` path, that sets the extension on outgoing video packets.
  - Chrome, Edge, Safari and Firefox all embed libwebrtc. Whether Firefox and Safari actually *negotiate* this extension was not verified; test it per browser.
- **`RTCRtpReceiver.jitterBufferTarget`** ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/RTCRtpReceiver/jitterBufferTarget), [spec](https://w3c.github.io/webrtc-pc/#dom-rtcrtpreceiver-jitterbuffertarget)):
  - It is the standard, receiver-side hint, in milliseconds from 0 to 4000.
  - Support: Chrome 124+, Firefox 115+, Safari 27+. It became Baseline in September 2026.
  - It only *influences* the target; the user agent clamps it to what network conditions allow.
  - Set it to 0 on both the audio and video receivers.

### How to measure

- **Browser `getStats()`, `inbound-rtp`** (MDN `RTCInboundRtpStreamStats`):
  - `jitterBufferDelay/jitterBufferEmittedCount` gives the average time in the jitter buffer.
  - `jitterBufferTargetDelay` and `jitterBufferMinimumDelay` show the target the buffer is aiming for, compared with what the network alone would require.
  - `totalDecodeTime`, `totalProcessingDelay` and `totalAssemblyTime` cover the rest of the receive path.
  - `totalInterFrameDelay` and `totalSquaredInterFrameDelay` measure frame-pacing jitter.
- **`requestVideoFrameCallback`** ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/HTMLVideoElement/requestVideoFrameCallback)), Baseline since October 2024 (Chrome 83, Firefox 132, Safari 15.4). For WebRTC sources its metadata adds:
  - `receiveTime`, when the last packet of the frame arrived;
  - `captureTime`, estimated from RTCP sender reports;
  - `rtpTimestamp`;
  - `expectedDisplayTime` and `processingDuration`.

  `expectedDisplayTime - receiveTime` is receive-to-photon, roughly. Matching `rtpTimestamp` to a host-side log of (frame number, RTP timestamp, input sequence number applied, encode start/end) gives the input-to-frame part.
- **End-to-end ground truth:** stamp the input sequence number into the emulated frame, for example a debug pixel strip read back with a canvas, or log it host-side against the RTP timestamp. Then compute `input event time → expectedDisplayTime` of the first frame that applied it. A high-speed camera pointed at the screen with a visible input LED is the external cross-check.

Measuring this against the budget is the map's open "Observability" item.

## Packaging cost in a Nix-built image

Measured with `nix path-info -S` on nixpkgs-unstable, 2026-10-22. Closure sizes include glibc, which is ~36 MiB on its own.

| Package | Version | Closure | License |
|---|---|---|---|
| `x264.lib` | 0-unstable-2025-09-10 | 38.8 MiB | GPL-2.0-or-later |
| `libvpx` | 1.17.0 | 55.6 MiB | BSD-3-Clause |
| `openh264` | 2.6.0 | 48.3 MiB | BSD-2-Clause |
| `svt-av1` | 4.1.0 | 43.9 MiB | BSD-3-Clause (+ AOM patent license) |
| `libaom` | 3.15.1 | 58.1 MiB | |
| `libopus` | 1.6.1 | 37.0 MiB | BSD-3-Clause |
| x264 + libvpx + libopus combined | | **58 MiB** | |
| GStreamer core + base + good + bad + ugly + libnice | 1.28.7 | **828 MiB** | LGPL (plugins vary) |

Pion is pure Go. The cgo wrappers need `pkg-config` in `nativeBuildInputs` and the libraries in `buildInputs` of `buildGoModule`, with `CGO_ENABLED=1`. Every wrapper above uses `#cgo pkg-config:`, so nixpkgs' pkg-config hook resolves them with no `PKG_CONFIG_PATH` handling.

## Open points for later tickets

- **#7 (dual-screen layout):** a single stacked 256x384 track, encoded at 2x, keeps the two screens frame-synchronous and needs one encoder. The browser can crop or rearrange with CSS or canvas for any layout. Two tracks would need cross-track sync, which WebRTC does not guarantee per frame.
- **#8 (frontend protocol):** the protocol rides on one `RTCPeerConnection` holding a video track, an audio track, an unreliable input DataChannel and optionally a reliable control channel. Signalling (SDP and ICE exchange) is outside WebRTC and still needs a transport. WHIP/WHEP-style HTTP POST is the lightest; Pion ships a `whip-whep` example.
- **#11 (input):** the input channel should send idempotent full-state snapshots with sequence numbers over an unordered, zero-retransmit channel. Touch coordinates should be in DS screen space (256x192), so they do not depend on the encode scale. Mic belongs on a media track, not the DataChannel.
- **Licensing:** if H.264 is wanted, prefer OpenH264 (BSD) over x264 (GPL) unless the published image is allowed to be GPL.

## Sources

- [MDN: Codecs used by WebRTC](https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats/WebRTC_codecs); [RFC 7742](https://datatracker.ietf.org/doc/html/rfc7742); [RFC 7874](https://datatracker.ietf.org/doc/html/rfc7874)
- [MDN: `RTCRtpReceiver.jitterBufferTarget`](https://developer.mozilla.org/en-US/docs/Web/API/RTCRtpReceiver/jitterBufferTarget) and BCD `api.RTCRtpReceiver.jitterBufferTarget`
- [MDN: `requestVideoFrameCallback`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLVideoElement/requestVideoFrameCallback) and BCD; [MDN: `RTCInboundRtpStreamStats`](https://developer.mozilla.org/en-US/docs/Web/API/RTCInboundRtpStreamStats)
- [libwebrtc playout-delay extension](https://webrtc.googlesource.com/src/+/main/docs/native-code/rtp-hdrext/playout-delay/README.md); [libwebrtc `timing.cc`](https://webrtc.googlesource.com/src/+/HEAD/modules/video_coding/timing/timing.cc)
- `github.com/pion/webrtc/v4@v4.2.25` source (`mediaengine.go`, `interceptor.go`, `datachannel.go`, `track_local_static.go`); `github.com/pion/rtp@v1.10.5` (`playoutdelayextension.go`); `github.com/pion/interceptor@v0.1.49` (`pkg/`)
- `github.com/pion/mediadevices@v0.10.1` source (`pkg/codec/{x264,vpx,openh264,svtav1,opus}`) and [README](https://github.com/pion/mediadevices)
- [x264 `common/base.c`](https://github.com/mirror/x264/blob/master/common/base.c) and [`x264.h`](https://github.com/mirror/x264/blob/master/x264.h)
- [GStreamer webrtc plugin docs](https://gstreamer.freedesktop.org/documentation/webrtc/index.html); [go-gst README](https://github.com/go-gst/go-gst)
- [cloud-game `config.yaml`](https://github.com/giongto35/cloud-game/blob/master/pkg/config/config.yaml), [`go.mod`](https://github.com/giongto35/cloud-game/blob/master/go.mod) and [README](https://github.com/giongto35/cloud-game)
- nixpkgs-unstable (2026-10-22): package versions, `meta.license` and closure sizes from `nix eval` and `nix path-info`
- Local benchmark: FFmpeg 9.0.2 `-benchmark`, script below

### Benchmark script

```bash
# s1.yuv = 600 frames of testsrc2 at 256x384@60, yuv420p; s2/s3 = nearest-neighbour 2x/3x
enc() { name=$1; s=$2; shift 2
  ffmpeg -nostats -benchmark -f rawvideo -pix_fmt yuv420p -s ${SZ[$s]} -r 60 -i s$s.yuv \
    -threads 1 -b:v ${BR[$s]} "$@" -f null - 2>&1 | grep -o 'utime=.*rtime=[0-9.]*s'; }
enc x264-ultrafast 2 -c:v libx264 -preset ultrafast -tune zerolatency -profile:v baseline
enc vp8            2 -c:v libvpx -deadline realtime -cpu-used 8 -lag-in-frames 0
enc svtav1         2 -c:v libsvtav1 -preset 12 -svtav1-params rtc=1:pred-struct=1:rc=2:lp=1
enc aom            2 -c:v libaom-av1 -usage realtime -cpu-used 8 -lag-in-frames 0
```
