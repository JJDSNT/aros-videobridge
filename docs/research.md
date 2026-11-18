# Research notes and source evidence

This document separates **observed source contracts**, **upstream claims**, and **open verification items**. Links are to the inspected upstream paths.

## AROS VHI and UVC

1. [VHI public header](https://github.com/aros-development-team/AROS/blob/master/compiler/include/vhi/vhi.h): method dispatch; `VHI_DIGITIZE_PICTURE`; RGB24/YUV/JPEG image representation; optional custom allocation callbacks. No timestamps or audio in `vhi_image`.
2. [VHI library definition](https://github.com/aros-development-team/AROS/blob/master/workbench/libs/vhi/vhi.conf): one exported `vhi_api(method, submethod, attr, errcode, vhi_handle)` entry point.
3. [UVC public capture contract](https://github.com/aros-development-team/AROS/blob/master/compiler/include/devices/uvc_capture.h): `UVCM_COUNT/INFO/GRAB/MODECOUNT/MODEINFO/DEVICEINFO/RATES/GRABRATE`; optional VHI-specific UVC extensions; documented RGB24 buffer contract and failure conditions.
4. [UVC class implementation](https://github.com/aros-development-team/AROS/blob/master/rom/usb/classes/video/uvc.class.c): USB streaming descriptor parsing, frame-mode selection, MJPEG/YUY2/grayscale handling; class-level capture logic, not proof of a generic streaming VHI API.
5. [cAMIra VideoHardware.h](https://github.com/aros-development-team/contrib/blob/master/MultiMedia/camira/VideoHardware.h): a separate VHI process, `MsgPort` messages, and an existing capture consumer. Useful for testing an unmodified VHI client.

**Caution:** The VHI method header and UVC extensions are not a full audit of driver installation, implementation, lifecycle, or target availability. Inspect actual driver sources and test builds before making compatibility claims.

## FFmpeg in AROS

[FFmpeg 8.1.2 AROS patch](https://github.com/aros-development-team/contrib/blob/master/MultiMedia/libs/ffmpeg/ffmpeg-8.1.2-aros.diff) exists under `contrib/MultiMedia/libs/ffmpeg`. Its inspected changes include AROS-specific socket handling (`CloseSocket`) and VA-API/AROS configuration and device support.

This is **positive evidence of a porting effort**, not proof of an installed FFmpeg executable, successful RTSP/RTMP/SRT support, working encoders, or m68k performance. Determine supported build targets, executable/library packaging, codecs, protocol enablement and test results.

## Copperline versus NT-Toaster versus OBS plugin

These are distinct layers and must not be conflated.

- [Video-Toaster-Copperline](https://github.com/jfoust2/Video-Toaster-Copperline) emulates the original Amiga/Video Toaster hardware. Its [Toaster working specs](https://github.com/jfoust2/Video-Toaster-Copperline/tree/main/docs/toaster) derive from the NewTek source release. This is a reference for Toaster behavior, not an AROS streaming API.
- [NT-Toaster README](https://github.com/jfoust2/NT-Toaster/blob/main/README.md) describes `libtoastercard`, a C99 port of the Copperline card model, and a separate native Windows `shell/` that integrates files, cameras, network streams, desktop and NDI.
- [NT-Toaster shell README](https://github.com/jfoust2/NT-Toaster/blob/main/shell/README.md) states that each input is an FFmpeg child process producing raw 752x480 RGBA at 29.97 fps over a pipe and 48 kHz stereo audio over a local TCP socket. The engine runs at 59.94 fields/s. Output recording/streaming uses a separate FFmpeg process. The README reports a tested local UDP streaming loop and NDI in/out; it does **not** establish individual end-to-end validation of all listed protocols.
- [obs-nt-toaster README](https://github.com/jfoust2/obs-nt-toaster/blob/main/README.md) says the plugin delegates general recording, streaming and audio to OBS. Its initial version built and loaded but had not been functionally exercised.
- [obs-nt-toaster source](https://github.com/jfoust2/obs-nt-toaster/blob/main/src/toaster-source.c) renders four selected OBS sources into 752x480 textures, reads pixels back, advances the card engine at 59.94 fields/s and exposes Program as an OBS video source. This demonstrates an **adapter boundary**, not an AROS-compatible plugin or a zero-copy path.

## What to reuse conceptually

- Distinguish decoder/capture adapters from the video-processing engine.
- Keep the engine independent of RTSP/RTMP/SRT and of OBS-specific APIs.
- Allow asynchronous I/O and bounded queues to isolate stalls.
- Treat publishing and ingest as separate directions.
- Keep frame-rate conversion and field/frame timing explicit; Toaster's 59.94 field cadence must not be assumed to be a universal VHI requirement.

## Architectural references: media routing and generic I/O

These projects are **architectural references**, not dependencies, porting commitments, or evidence that their implementations run on AROS. The VideoBridge scope remains **a reusable, lightweight audiovisual input/output infrastructure**; effects, compositing, timeline editing, and application-specific rendering belong to consumers built above it.

### MediaMTX — stream publication and distribution

- Repository: https://github.com/bluenviron/mediamtx
- Implementation points: [path lifecycle](https://github.com/bluenviron/mediamtx/blob/main/internal/core/path.go), [stream routing](https://github.com/bluenviron/mediamtx/blob/main/internal/stream/stream.go), [reader queues](https://github.com/bluenviron/mediamtx/blob/main/internal/stream/reader.go).
- **Observed:** a path associates a publisher/source, stream and readers. Readers register callbacks for media formats and have bounded ring buffers; a full reader queue results in dropped outbound units rather than unlimited buffering. MediaMTX routes media across supported network protocols without generally requiring video decoding.
- **Relevant idea:** named streams, independent publisher/subscriber lifecycles, one-to-many delivery, per-subscriber isolation and explicit backpressure.
- **Do not import:** the Go runtime, complete network-server stack, authentication/control/metrics subsystems, or an assumption that all inputs are network streams.
- **OBS interoperability, documented upstream:** [OBS publishing via RTMP or WHIP](https://github.com/bluenviron/mediamtx/blob/main/docs/3-publish/19-obs-studio.md); [OBS reading via RTSP Media Source](https://github.com/bluenviron/mediamtx/blob/main/docs/4-read/11-obs-studio.md). These demonstrate MediaMTX/OBS compatibility, **not** an AROS VideoBridge publisher.

### GPAC — generic I/O contracts and packet exchange

- Repository: https://github.com/gpac/gpac
- Reference: https://wiki.gpac.io/Filters/filters_general/
- **Upstream design:** a filter-based media framework whose components expose input/output capabilities, connect media endpoints, and exchange timestamped packets. Sources, sinks and optional transformations can share a common connection model.
- **Relevant idea:** capability negotiation, packet ownership/lifetime, format changes, and separation of raw-frame delivery from compressed-packet passthrough.
- **Do not import:** a general-purpose processing graph, full multimedia runtime, built-in effects or an assumption that GPAC's scheduler and memory model fit low-memory m68k. A VideoBridge input/output contract can remain much smaller than a generic filter graph.

### libobs — application-facing source/output boundaries

- Repository: https://github.com/obsproject/obs-studio (libobs)
- **Relevant idea:** distinguish sources, outputs and optional encoders; expose encoded and uncompressed media paths without making the I/O layer itself an effects engine.
- **Boundary:** OBS rendering, scene composition, GPU and desktop dependencies are outside VideoBridge. An OBS host adapter and an open-protocol network publisher are different integration choices.

### LIVE555 — focused RTP/RTSP transport

- Repository: https://github.com/rgaufman/live555
- Official project: http://www.live555.com/liveMedia/
- **Relevant idea:** separate RTP/RTSP sessions, transport packetization and media-source abstractions. Useful when evaluating an optional RTSP/RTP adapter.
- **Boundary:** not a replacement for a generic local-device/file/application I/O core; assess license, target portability, runtime and memory before considering any code reuse.

### PipeWire — local-device and inter-application media routing

- Repository: https://github.com/PipeWire/pipewire
- **Relevant idea:** connecting device and application endpoints, media format negotiation, and coordinating buffers between independent participants.
- **Boundary:** its Linux-centric process integration and supporting infrastructure should not become prerequisites for an AROS implementation.

### VideoBridge design implications (proposals, not settled ABI)

1. Keep a **small transport-agnostic core** for stream identity, endpoint registration, subscriptions, media format descriptors, lifecycle and bounded delivery.
2. Support **raw frames and encoded packets** explicitly. Preserve packet passthrough when source and sink capabilities permit; do not imply universal transcoding or zero-copy.
3. Make capture (VHI/UVC), file I/O, network protocols, host/OBS interoperability and codecs **optional adapters**. AROS Exec MsgPorts/tasks may carry control and notifications; payload ownership, copying and task affinity need separate contracts.
4. Bound memory in **bytes**, not only item counts; define slow-reader policies, buffer ownership, cancellation, timestamp/clock domains and discontinuities. Prefer measured m68k budgets over desktop assumptions.
5. Permit an FX application to subscribe to a stream and publish its processed result as a **new stream**; processing remains outside the VideoBridge core.
6. Keep MediaMTX as the leading **scope/routing reference**, GPAC as a **generic I/O contract reference**, and libobs/LIVE555/PipeWire as narrower complementary references. This ranking is an architectural assessment, not a benchmark.

**Open validation:** inspect upstream buffer/refcount and scheduling behavior in depth; measure AROS m68k frame/packet memory, copy counts and throughput; prove an actual AROS publisher-to-MediaMTX-to-OBS path; and check protocol/codec availability per target before selecting a first adapter.

## Open questions requiring code and runtime evidence

1. Which actual VHI driver in AROS consumes UVC methods, and how are its buffers owned and released?
2. Does the VHI digitize call block for a new frame, return a cached frame, or vary by driver? How are concurrent callers handled?
3. What is the exact availability of the FFmpeg 8.1.2 port on m68k, x86 and AArch64? Which protocols and encoders are enabled?
4. Which AROS network/socket facilities and resolver services are needed by the selected protocol? Avoid stack changes without demonstrated gaps.
5. What minimum buffer count and conversion strategy sustain target frame rates on low-memory m68k?
6. Which OBS ingestion route works with an AROS publisher without requiring a custom OBS plugin?
7. What license and runtime constraints apply to NDI and any adapted code?
8. What audio synchronization is needed for the first actual Video Toaster use case?

## Attribution and provenance

NewTek authored the original Video Toaster/Flyer and released source material later hosted by [Open Video Toaster / DiscreetFX](https://www.discreetfx.com/openvideotoaster.html). Copperline's Toaster fork and NT-Toaster independently implement and port hardware behavior; the OBS integration is a separate project. AROS VHI, UVC and cAMIra are independently developed AROS/Amiga components. Preserve copyright notices and upstream provenance in any future code adaptation.
