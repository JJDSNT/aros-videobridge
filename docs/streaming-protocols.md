# Streaming protocols: support strategy and priorities

**Status:** research-backed proposal, not an implementation or an AROS compatibility claim.  
**Scope:** both network sources **into** AROS and publishing AROS-produced video **to** OBS or other receivers.  
**Principle:** prioritize complete interoperable paths (transport + container + codec + receiver), not protocol names in isolation.

## 1. The key architectural constraint

OBS **Media Source** is backed by FFmpeg and can ingest supported network media; OBS also supports specific streaming outputs. An OBS source plugin is a different integration model from an OBS output plugin. Copperline's Toaster model, NT-Toaster's FFmpeg-based shell, and the OBS plugin are distinct pieces.

A stream is a stack:

```text
Capture/Toaster program -> frame format -> encoder -> container/packetization
                        -> transport -> receiver demux -> decoder -> OBS source

Network source -> transport -> demux -> decoder -> VHI-compatible frame source
```

A `udp://` URL alone does **not** imply that OBS understands arbitrary RGB frames. A tested combination might be **MPEG-TS over UDP** with a codec that both endpoints actually support.

## 2. Proposed priority matrix

| Priority | Candidate | AROS input | AROS output -> OBS | Rationale and dependencies |
| --- | --- | --- | --- | --- |
| **P0 — foundation** | **Raw framed video over TCP (custom host bridge)** | Optional | Yes, **with a host receiver/plugin** | Useful low-complexity diagnostic path avoiding on-AROS encoding. Not a standard OBS Media Source input. Must specify framing, pixel format, timestamp, backpressure, and receiver. |
| **P1 — first standard network path** | **MPEG-TS over UDP (unicast)** | Yes, if demux/decoder available | Yes, if encoder/mux available | Small transport dependency; easy LAN testing; no retransmission, no confidentiality. First *standard OBS Media Source* proof of concept. |
| **P1 — source ingest** | **RTSP/RTP, preferably RTSP over TCP initially** | Yes | Conditional; not a default OBS publishing target | Important for IP cameras. Session setup, authentication, RTP timestamps, codecs, reconnect, and optional UDP transport add complexity. |
| **P2** | **SRT + MPEG-TS** | Yes, if libsrt/FFmpeg enabled | Yes, if libsrt/FFmpeg enabled | Resilience to packet loss and jitter; useful beyond LAN. External libsrt availability and memory/CPU must be verified. |
| **P2** | **RTMP / RTMPS** | Conditional | Yes, generally via a receiving RTMP server or appropriate OBS Media Source | Widely used distribution workflow; RTMPS adds TLS; not a direct replacement for simple peer-to-peer LAN capture. |
| **P3** | **HTTP(S) media / HLS** | Yes, if appropriate demuxer/codec/TLS | Not a low-latency live sink by default | Valuable as remote source compatibility, but segment buffering and latency are poor fits for responsive switching. |
| **P3 — optional** | **RIST + MPEG-TS** | Conditional | Conditional | OBS documents RIST as a supported resilient transport; requires verifying librist/FFmpeg availability on AROS. |
| **P4 — research** | **WebRTC / WHIP / WHEP** | Conditional, complex | Conditional, complex | Interactive latency is attractive, but ICE/STUN/TURN, DTLS/SRTP, signaling, codec negotiation and target footprint make it unsuitable as the first AROS transport. WHIP is an ingest/publishing convention, not evidence of generic WebRTC input support in OBS Media Source. |
| **Not initially** | **Generic raw UDP frames** | Custom only | Custom only | Without framing, sequencing, fragmentation, format negotiation and a receiver it is neither robust nor OBS-compatible. |
| **Not initially** | **Multicast UDP** | Deferred | Deferred | No first-use-case requirement; increases network and stack requirements. Start with unicast. |

**Important:** P0 raw TCP and P1 UDP/MPEG-TS are **alternative feasibility paths**, not an obligation to implement both. If native encoding is infeasible on the target, P0 plus a host-side FFmpeg/OBS adapter may be the viable first deliverable.

## 3. Direction-specific recommendation

### AROS -> OBS: first choose the encoding boundary

**Path A: encode on AROS.** VHI/Toaster frames -> codec -> MPEG-TS mux -> UDP sender -> OBS Media Source. Test a real encoder and OBS decoder first; do not assume H.264 encoding is practical on legacy m68k.

**Path B: encode on a host.** AROS frames -> bounded framed TCP transport -> host bridge -> FFmpeg/OBS. This is not directly OBS-native; a host application or OBS plugin is required. It can move expensive encoding off AROS and preserve a small AROS footprint.

**Path C: host-native OBS plugin.** A plugin ingests an explicitly versioned AROS transport and exposes Program as an OBS source. NT-Toaster's OBS plugin provides an architectural precedent, not a drop-in AROS implementation.

### OBS / IP camera -> AROS

For IP cameras, investigate RTSP/RTP (including RTSP interleaved over TCP). For OBS as a sender, prefer a verified OBS MPEG-TS-over-UDP or SRT output configuration and a corresponding AROS receiver. A decoded frame can be surfaced through a **virtual VHI device** while keeping the public VHI ABI unchanged.

## 4. Codec/container choices are separate decisions

- **MPEG-TS** is a container/transport stream, **not** a video codec.
- **H.264** offers broad interoperability but software encoding can be prohibitive on legacy systems.
- **MJPEG** may be useful for camera passthrough or simpler per-frame decoding/encoding, but bandwidth and actual MPEG-TS/OBS support for the selected path must be tested.
- **Raw RGB24/YUY2** avoids compression cost but consumes substantial bandwidth and memory; requires explicit framing or a suitable container and receiver.
- **AAC/PCM audio** is deferred until the video-only path works; later define shared clock origin, timestamps, drift correction and synchronization.

At 640x480x30 fps: RGB24 is ~27.6 MB/s (~221 Mb/s) **per stream** before headers; YUY2 is ~18.4 MB/s (~147 Mb/s). Four raw RGB24 inputs approach ~111 MB/s before copies and framing. These numbers make a raw network bridge inappropriate for some legacy hardware and networks; they are not performance benchmarks.

## 5. Required protocol contract (independent of implementation)

Each adapter should document:

1. **Direction and role:** client/server, sender/receiver, listener/caller, bind address and port.
2. **Media negotiation:** container, codec, pixel format, resolution, frame/field rate, optional audio.
3. **Timing:** monotonic timestamps, stream clock, jitter buffering, frame dropping, field cadence.
4. **Memory:** bounded queues, max frame size, allocation ownership, copy count, disconnect cleanup.
5. **Recovery:** timeouts, reconnect/backoff, discontinuity, resolution changes, malformed packets.
6. **Security:** trusted LAN vs untrusted network; TLS/encryption, credentials, exposed listening sockets.
7. **Dependencies:** target architecture, FFmpeg build options, optional libraries, codec licensing.
8. **Validation:** OBS version, exact URL/configuration, duration, latency, CPU, memory and failure tests.

Do not place streaming protocols directly inside `vhi.library`. Keep VHI compatibility at the source adapter boundary and transport-specific code in optional modules.

## 6. Concrete test gates

### Gate A — inventory and native feasibility

- Identify working FFmpeg binaries/libraries per AROS architecture, not merely a patch file.
- Capture `ffmpeg -protocols`, `-demuxers`, `-muxers`, `-decoders`, `-encoders`, and `-buildconf` from the actual target.
- Verify UDP/TCP sockets, timing, and any TLS/libsrt/librist dependencies on that target.
- Test VHI/UVC buffer ownership and whether the source is pull-based.
- Establish a measured target budget for m68k RAM, CPU and network throughput.

**Exit:** choose Path A or Path B based on measurements.

### Gate B — standard OBS receive path

- Publish a short, known-good MPEG-TS-over-UDP stream from a host reference FFmpeg build.
- Configure OBS Media Source to receive it and record exact codec/container/URL options.
- Repeat with an AROS sender **only if** a working encoder/mux is demonstrated.
- Test packet loss, disconnect, restart and at least 10 minutes of stable reception.

**Exit:** reproducible AROS -> OBS proof of concept or a documented infeasibility result.

### Gate C — low-cost bridge fallback

- Define a small versioned frame header and TCP framing, with maximum lengths and timestamp semantics.
- Build a host receiver that forwards to OBS/FFmpeg.
- Test source stall, slow receiver, bounded memory, and graceful shutdown.
- Compare end-to-end latency, CPU, copies and bandwidth against Gate B.

**Exit:** a measured recommendation for the smallest reliable AROS output path.

### Gate D — input and extended protocols

- Add RTSP camera ingestion once decoding and virtual VHI work.
- Add SRT when libsrt availability and recovery requirements justify it.
- Consider RTMP(S), HLS, RIST and WebRTC only for demonstrated use cases and measured budgets. NDI is excluded from proposed implementation because it is proprietary.

## 7. Evidence and verification limits

### OBS official documentation

- [OBS SRT Protocol Streaming Guide](https://obsproject.com/kb/srt-protocol-streaming-guide): Media Source/VLC Source SRT ingest, caller/listener roles, and MPEG-TS-based output protocols including UDP/TCP/RTP.
- [OBS Streaming With SRT or RIST Protocols (project wiki mirror)](https://github-wiki-see.page/m/obsproject/obs-studio/wiki/Streaming-With-SRT-or-RIST-Protocols): RIST and SRT operation, buffering and latency tuning. Treat version and build-specific claims as requiring current verification.

### FFmpeg upstream

- [FFmpeg protocols](https://ffmpeg.org/ffmpeg-protocols.html): RTSP demux/mux distinction, RTP/UDP/TCP/RTMP/SRT behavior and optional dependencies.
- [FFmpeg general support matrix](https://ffmpeg.org/general.html): protocol and external-library support; upstream capabilities are **not** proof of AROS build capabilities.

### AROS and reference projects

- [AROS FFmpeg 8.1.2 patch](https://github.com/aros-development-team/contrib/blob/master/MultiMedia/libs/ffmpeg/ffmpeg-8.1.2-aros.diff): AROS port adaptation; build/feature status unverified.
- [AROS VHI header](https://github.com/aros-development-team/AROS/blob/master/compiler/include/vhi/vhi.h): legacy capture API.
- [AROS UVC capture header](https://github.com/aros-development-team/AROS/blob/master/compiler/include/devices/uvc_capture.h): device/mode/frame capture.
- [NT-Toaster shell](https://github.com/jfoust2/NT-Toaster/blob/main/shell/README.md): FFmpeg-backed source and output adapters, raw host pipes, audio socket and NDI.
- [NT-Toaster OBS plugin](https://github.com/jfoust2/obs-nt-toaster): OBS source adapter precedent.

## 8. Decision summary

**Initial support target:** unicast **UDP/MPEG-TS** as the first standard OBS interoperability experiment; **raw framed TCP to a host bridge** as a parallel, low-encoding-cost feasibility option; **RTSP input** as the first camera-network use case. **SRT** follows after a verified libsrt/FFmpeg target build. The remaining open-protocol candidates are optional and demand-driven. NDI may be mentioned as a factual feature of upstream NT-Toaster but is not a VideoBridge implementation target.

No public AROS/VHI API changes should be proposed until these gates produce reproducible evidence.
