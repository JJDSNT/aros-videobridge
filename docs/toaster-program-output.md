# Video Toaster Program Output: architectural options and decision rationale

**Status:** design research and decision record; no implementation selected.  
**Purpose:** preserve the full reasoning behind candidate output mechanisms so future development agents can revisit them when the AROS Video Toaster integration source becomes available.

## 1. Context and assumptions

The project takes as its working premise that both the AROS Video Toaster initiative and the Copperline/NT-Toaster family are rooted in the **Open Video Toaster / original NewTek software**. The shared provenance does **not** imply the same video-output implementation: Copperline emulates the Toaster hardware and NT-Toaster provides a modern host shell, while the actual AROS Toaster output code has **not yet been inspected**.

The AROS VideoBridge mission is broader than a Toaster-specific output patch: bring modern camera, file, streaming and external-application I/O to AROS, allowing Video Toaster and other native applications to consume and produce video without embedding protocol logic into VHI.

**Known:** AROS has VHI capture interfaces and UVC-related interfaces; NT-Toaster uses external FFmpeg and NDI output adapters; obs-nt-toaster is an OBS source plugin.  
**Unknown:** the AROS Toaster Program output path, whether it is bitmap/RTG, an internal frame buffer, a hardware-facing driver abstraction, or something else. No specific AROS Toaster output implementation is asserted here.

## 2. Source-backed precedent: what NT-Toaster actually does

[NT-Toaster standalone shell documentation](https://github.com/jfoust2/NT-Toaster/blob/main/shell/README.md) describes four FFmpeg-backed input processes delivering 752x480 RGBA frames at 29.97 fps over pipes, with 48 kHz stereo audio on local TCP sockets. The Toaster card engine advances at 59.94 fields per second.

For output, the standalone shell feeds **Program frames to a separate FFmpeg process through stdin**, while mixed audio is delivered over a socket and queue to avoid blocking the mixer. The documented recording mode is MP4 H.264/AAC (640x480); streaming uses H.264/AAC (960x720, approximately 3 Mbit/s) with RTMP, SRT, UDP or RTSP destinations. The shell also offers an independently loaded NDI 6 runtime for Program publication. The project reports a successful NDI send/receive loop and a local UDP streaming test; that is not proof every advertised protocol was individually tested.

The [OBS integration](https://github.com/jfoust2/obs-nt-toaster/blob/main/src/toaster-source.c) instead registers an OBS **input video source**. It supplies four OBS-selected sources to the card engine, retrieves Program RGBA, uploads it to an OBS texture, and lets OBS handle encoding, recording and broadcasting. This is **not** an OBS output plugin and is **not** a ready-made AROS integration. The plugin README describes early build/load success without complete functional exercise.

**Transferable insight:** separate the engine's Program frame production from optional output consumers. The transport/encoder boundary is a design choice, not an intrinsic property of the Toaster engine.

## 3. Candidate AROS output interception mechanisms

The alternatives below are hypotheses to evaluate against the real AROS port, **not verified existing features**.

| Option | How it would work | Strengths | Costs and risks | When to prefer |
| --- | --- | --- | --- | --- |
| **A. Display/RTG bitmap capture** | Read the final image from the existing AROS display surface, bitmap or window | Minimal modification to Toaster internals; may work with opaque application | Readback, copies, format conversion, window occlusion/visibility, timing and tearing; screen image may include UI rather than clean Program | When the port exposes no earlier Program frame hook |
| **B. Program frame callback** | Hook the engine's point of completed Program frame production and dispatch to consumers | Clean Program signal, precise timing, avoids screen scraping, simple source-to-sink separation | Requires access to and modification of the actual port; callback must not block rendering; frame lifetime must be defined | **Preferred candidate** if an accessible frame boundary exists |
| **C. Shared/double/triple frame buffers** | Producer and consumers exchange references to versioned buffers | Potentially fewer copies; can isolate producers and consumers | Ownership, task lifetime, cache coherency, synchronization, fragmentation, stale pointers, architecture-specific memory visibility | When measured copy cost justifies complexity and same-address-space constraints are satisfied |
| **D. Virtual hardware-facing output driver** | Replace/emulate the output interface the original Toaster expects | May preserve original application flow; can minimize modifications to NewTek-derived code | Depends entirely on how the AROS port replaces hardware access; may be tied to undocumented card/field semantics | When the port already has a clean hardware adapter boundary |
| **E. Display-only integration** | Publish only the rendered AROS window/screen | Fast diagnostic fallback | Not suitable as a general clean Program signal; may miss frame cadence, interlace, and audio | Debugging or initial feasibility only |

**Selection rule:** first identify the *earliest reliable completed Program frame* in the existing AROS pipeline. Prefer B if available; consider D if the port is structured around a hardware adapter; use A/E as fallback; introduce C only after profiling. Do not prematurely mandate one choice.

## 4. Proposed logical boundaries (not a public ABI)

```text
VHI/UVC camera --+
network decoder -+--> VideoSource adapters --> Toaster / video processor
media file ------+                                   |
synthetic -------+                                   v
                                            Program frame boundary
                                                       |
                                            bounded fan-out / dispatch
                                                       |
                         +-----------------------------+------------------------+
                         |                             |                        |
                    local display                encoder / mux             host bridge
                    (AROS RTG)                  (file / stream)           (OBS / FFmpeg)
                                                                               |
                                                                          optional NDI
```

Candidate internal concepts:

- `VideoSource`: frame producer or pull adapter (VHI capture, decoded file/network, synthetic source).
- `ProgramFrame`: size-tagged frame descriptor with dimensions, pixel format, planes/strides, monotonic timestamp, duration or field phase, sequence, discontinuity, ownership/release contract.
- `VideoSink`: independently optional consumer (local display, file encoder, stream publisher, external bridge).
- `FrameDispatcher`: bounded queues, explicit drop/backpressure policy, per-sink isolation.

These are **conceptual names**, not promised C symbols or library names. A new public library should not be introduced before a real use case demonstrates the need. Do not change the layout of legacy `struct vhi_image` or the existing `vhi_api` entry point merely to support these concepts.

## 5. Distinguish Program output from preview and UI

The original Video Toaster distinguishes **Program** (on-air result) from **Preview** (next/composition preview). A desktop window is not automatically the clean Program output. An AROS adapter must determine:

- whether Program and Preview are separate frame surfaces;
- whether overlays, switcher controls or the application UI contaminate display capture;
- whether effects are produced at field cadence (e.g. 59.94 fields/s) rather than only full frames;
- whether color representation is composite/Toaster-specific, chunky RGB, planar, YUV or RTG-native;
- how audio follows Program and whether transition/mixer state is available.

Do not assume NT-Toaster's 752x480 RGBA host format or its 59.94-field cadence is automatically the native format of the AROS port. Conversion should occur at explicit boundaries.

## 6. Output destinations and codec placement

### Path 1 — AROS-local encoder and network publisher

```text
Program -> encoder -> muxer -> network sink -> OBS Media Source
```

Pros: standard interoperability, no custom host receiver. Cons: codec CPU/RAM, muxing and protocol dependencies on AROS. The [AROS FFmpeg 8.1.2 patch](https://github.com/aros-development-team/contrib/blob/master/MultiMedia/libs/ffmpeg/ffmpeg-8.1.2-aros.diff) demonstrates a porting effort, **not** a verified executable, codec availability or performance across targets. UDP/MPEG-TS is the initial standard proof-of-concept candidate; SRT and RTMP(S) require further dependency validation.

### Path 2 — AROS to external host bridge

```text
Program -> bounded framed transport -> host receiver -> FFmpeg / OBS
```

Pros: moves heavy encoding to a modern host, modular, avoids assuming H.264 on m68k. Cons: bandwidth and frame-copy costs; requires receiver; custom raw TCP is **not** directly readable by OBS Media Source. Must define versioned framing, size bounds, pixel format, timing, reconnect, security, and flow control.

### Path 3 — host-side OBS source plugin

```text
Program -> explicit host transport -> OBS source plugin -> OBS encoder/output
```

Pros: OBS handles publication, possible low latency and integration with scenes. Cons: plugin development/maintenance and OBS API compatibility; host transport still needed. NT-Toaster's plugin is an architectural precedent, not reusable without adaptation.

### Path 4 — NDI publisher

```text
Program -> NDI adapter -> NDI-aware receiver / OBS plugin
```

Pros: production-oriented discovery and interoperability. Cons: NDI runtime, license/redistribution and platform support, CPU/RAM and network bandwidth. Treat as optional and do not assume an NDI SDK/runtime works on AROS m68k.

**Crucial distinction:** a standard network **protocol** is not itself a frame output abstraction. Define the clean Program frame boundary before committing to UDP, SRT, RTMP or NDI.

## 7. Legacy m68k and resource constraints

Raw 640x480 RGB24 is 921,600 bytes per frame. At 30 fps it requires approximately 27.6 MB/s (221 Mbit/s) before overhead; YUY2 at the same geometry requires about 18.4 MB/s (147 Mbit/s). Four such RGB sources total about 110.6 MB/s. An eight-frame RGB24 queue occupies ~7.0 MiB, excluding decoder, network and application memory.

These are **arithmetic estimates**, not measured AROS throughput. They argue for:

- optional sinks and codecs, not a monolithic streaming dependency;
- very small bounded queues, potentially one/two frames on constrained targets;
- measured conversion and copy count; possible compressed-frame passthrough where compatible;
- asynchronous sink workers that cannot block the Program producer;
- configurable resolution/frame rate and graceful frame dropping;
- separate profiles for legacy m68k and less constrained AROS targets, if justified by build and runtime evidence.

A raw host bridge avoids local video encoding but does **not** eliminate raw transfer bandwidth. The first prototype should quantify that tradeoff rather than assume the bridge is universally cheaper.

## 8. Decision matrix

| Criterion | A: bitmap | B: callback | C: shared buffers | D: virtual output |
| --- | --- | --- | --- | --- |
| Minimal access to Toaster internals | High | Low | Medium | Depends on port |
| Clean Program without UI | Uncertain | Strong | Strong if hooked correctly | Depends on emulation |
| Low copy potential | Weak | Medium/strong | Strong | Depends |
| Complexity | Low initially | Medium | High | Potentially high |
| Frame/field timing fidelity | Weak/uncertain | Strong | Strong if metadata retained | Potentially strong |
| Low-memory suitability | Copies costly | Good with bounded buffers | Risky without careful design | Unknown |
| Recommended status | Fallback | **Preferred if available** | Profile-driven optimization | Investigate actual port |

This matrix is qualitative and must be revised after the port's actual frame path and target performance are known.

## 9. Evidence-gated investigation when AROS Toaster code becomes available

1. Locate its source repository, build configuration and original NewTek-derived files; preserve copyright/license provenance.
2. Trace capture and input selection: VHI driver calls, camera modes, source frame lifetime and conversion.
3. Trace every Program/Preview production and presentation function, including display bitmap/RTG and any hardware-adapter entry points.
4. Identify the first complete Program frame, its pixel/field representation, timing, ownership and task context.
5. Map audio/mixer output separately, including transitions and clock source.
6. Prototype the smallest **nonblocking** output hook with a synthetic/local sink; keep VHI ABI unchanged.
7. Compare bitmap capture, callback and virtual-driver hooks with measured frame correctness, latency, copies, CPU and peak RAM.
8. Select between local encoding and a host bridge using actual FFmpeg/codec support and m68k/network measurements.
9. Validate OBS end to end, including disconnect, recovery, resolution changes and sustained output.
10. Document the final choice in an ADR with rejected alternatives and the evidence that justified rejection.

**Stop condition:** if no clean Program boundary is found, document the limitation before inventing a new driver/API.

## 10. Relationship to other project documents

- [Architecture](architecture.md) — source/frame/sink boundaries.
- [Research](research.md) — upstream source-code evidence and verification gaps.
- [Streaming protocols](streaming-protocols.md) — transport priorities, codec/container separation, OBS ingest and output paths.
- [Roadmap](roadmap.md) — implementation and validation gates.

This document deliberately records **alternatives and reasoning**, rather than issuing a premature implementation instruction.

## 11. Attribution and primary references

- [Open Video Toaster / DiscreetFX](https://www.discreetfx.com/openvideotoaster.html) — original NewTek source distribution.
- [Copperline Toaster research](https://github.com/jfoust2/Video-Toaster-Copperline/blob/main/docs/toaster/README.md) — original source provenance and hardware model.
- [NT-Toaster standalone shell](https://github.com/jfoust2/NT-Toaster/blob/main/shell/README.md) — Program-to-FFmpeg stdin, audio socket, NDI, recording and streaming.
- [obs-nt-toaster source](https://github.com/jfoust2/obs-nt-toaster/blob/main/src/toaster-source.c) — OBS source integration and Program texture.
- [AROS VHI header](https://github.com/aros-development-team/AROS/blob/master/compiler/include/vhi/vhi.h) — capture ABI.
- [AROS UVC capture interface](https://github.com/aros-development-team/AROS/blob/master/compiler/include/devices/uvc_capture.h) — UVC capture and optional VHI extensions.
- [AROS FFmpeg port patch](https://github.com/aros-development-team/contrib/blob/master/MultiMedia/libs/ffmpeg/ffmpeg-8.1.2-aros.diff) — existing porting work.

Before copying upstream code, inspect applicable licenses and retain all original notices and credits.
