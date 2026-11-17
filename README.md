# AROS VideoBridge

**Research and architecture project — no implementation is claimed yet.**

AROS VideoBridge explores a modular, reusable audiovisual input/output infrastructure for AROS. It aims to connect cameras, media files, network streams, processing applications, and output destinations without coupling applications to specific codecs or transports.

**Concrete motivating examples:** an AROS **Video Toaster** could receive camera, file, and network sources and publish its clean Program output; **OBS Studio** could receive an AROS-produced stream through a compatible network format or a host-side adapter. These are motivating integration scenarios, not implemented features. The architecture should also support other AROS multimedia applications.

## Design intent

- Preserve the existing VHI driver/API contract and its value for native camera capture.
- Treat VHI as one capture interface, **not** as an encoder, streaming server, or mandatory output API.
- Introduce independently deployable source, frame-transport, and sink adapters where justified by evidence.
- Keep decoding, encoding, network transport, and device-specific code outside the minimal shared frame contract.
- Support low-memory m68k configurations: optional components, bounded queues, controlled allocations, explicit ownership, and measured copies.
- Avoid coupling the architecture to the NewTek application or to OBS internals. OBS interoperability can initially use standard network or capture interfaces.
- Credit the original projects and distinguish demonstrated capabilities from proposed ones.

## Reusable media pipeline

The intended architecture separates **source adapters**, **demuxers/decoders**, **format conversion and processing**, **encoders/muxers**, **transports**, and **sinks**. A pipeline should bypass unnecessary decoding or re-encoding when a compatible compressed stream can be passed through. Formats, timestamps, buffer ownership, capability negotiation, and backpressure belong to the minimal shared contracts; heavyweight codecs and protocols remain optional.

See [Media pipeline design](docs/media-pipeline.md) for the proposed boundaries, low-memory constraints, and integration scenarios. An OpenTimelineIO-style editing timeline is a potential *client* of media access, not a VideoBridge core responsibility. Display management and desktop extension remain outside this project's scope.

## Reference implementations

| Project | Relevant lessons |
| --- | --- |
| [AROS VHI API](https://github.com/aros-development-team/AROS/blob/master/compiler/include/vhi/vhi.h) | Existing method-based capture contract, image types, and custom allocation hooks |
| [AROS UVC capture interface](https://github.com/aros-development-team/AROS/blob/master/compiler/include/devices/uvc_capture.h) | Device discovery, negotiated modes, frame intervals, RGB24 capture |
| [AROS cAMIra](https://github.com/aros-development-team/contrib/tree/master/MultiMedia/camira) | AROS-native VHI client with a separate process and Exec message ports |
| [AROS FFmpeg port patch](https://github.com/aros-development-team/contrib/blob/master/MultiMedia/libs/ffmpeg/ffmpeg-8.1.2-aros.diff) | Evidence of an FFmpeg 8.1.2 adaptation, **not** proof that every protocol or codec is usable |
| [Video-Toaster-Copperline](https://github.com/jfoust2/Video-Toaster-Copperline) | Hardware-accurate Toaster emulation and historical video-path research |
| [NT-Toaster](https://github.com/jfoust2/NT-Toaster) | FFmpeg-backed file/camera/network inputs, output encoding, NDI integration |
| [obs-nt-toaster](https://github.com/jfoust2/obs-nt-toaster) | OBS-source adapter around a decoupled Toaster engine; early implementation, not fully functionally tested |
| [Open Video Toaster](https://www.discreetfx.com/openvideotoaster.html) | Original NewTek application/source distribution; not the subject of this infrastructure project |

## Documentation

- [Architecture and boundaries](docs/architecture.md)
- [Composable media pipeline](docs/media-pipeline.md)
- [Source-code findings and evidence](docs/research.md)
- [Streaming protocol support and priorities](docs/streaming-protocols.md)
- [Video Toaster Program output options and decision rationale](docs/toaster-program-output.md)
- [Incremental roadmap and acceptance criteria](docs/roadmap.md)

## Non-goals (initially)

- Reimplementing the NewTek Switcher or Flyer hardware model.
- Replacing existing VHI clients or breaking the VHI ABI.
- Embedding OBS or a complete FFmpeg stack inside vhi.library.
- Assuming all AROS targets have hardware acceleration, abundant RAM, or working codecs.

## Status

Architecture and research only. All implementation choices remain subject to code inspection, target-specific builds, and reproducible tests.

## Attribution

AROS and its contributors own the AROS code and interfaces; Copperline and NT-Toaster authors own their respective implementations; NewTek created the original Video Toaster and Flyer, whose software and materials are distributed through Open Video Toaster/DiscreetFX. This project documents interoperability ideas, not ownership of those works. Consult each upstream license before copying or adapting code.
