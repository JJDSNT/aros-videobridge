# Evidence-driven roadmap

No code or AROS ABI change is authorized by this roadmap alone. Each gate ends with measured evidence and an explicit decision.

## Gate 0 — Baseline and feasibility

- Locate the actual UVC-backed VHI driver implementation and trace `OPEN -> GET/SET -> DIGITIZE -> CLOSE`.
- Record driver buffer ownership, blocking semantics, frame formats, concurrency, unplug and cancellation behavior.
- Determine FFmpeg 8.1.2 build status per AROS target; enumerate actual enabled demuxers, protocols, decoders, encoders and dependencies.
- Establish a reproducible AROS build/test environment and baseline memory/CPU measurements.
- Identify one existing VHI client (e.g. cAMIra) and one OBS ingestion mechanism.

**Exit:** a verified dependency matrix and a minimal end-to-end diagram, with unsupported assumptions removed.

## Gate 1 — Virtual VHI source, no network

- Implement a minimal experimental VHI-compatible source fed by synthetic frames or a bounded local queue.
- Keep public `vhi.h` and `vhi.conf` unchanged.
- Exercise it from an existing VHI client without client modifications.
- Test frame ownership, source closure, repeated open/close, and low-memory failure.

**Exit:** a reproducible VHI client displaying generated frames with no leaks or unbounded queue growth.

## Gate 2 — Network/file ingestion

- Choose the smallest working decoder/protocol based on Gate 0, not on protocol popularity.
- Run blocking network/file I/O outside the consumer's rendering path.
- Implement bounded buffering, reconnect, timeout, format change and end-of-stream policies.
- Measure latency, frame drops, memory high-water mark and CPU.

**Exit:** a real stream/file appears to an unmodified VHI client; failures recover or terminate predictably.

## Gate 3 — OBS interoperability as output

- Export frames from a test producer or video consumer through a separate sink.
- First validate a supported OBS-ingest path; avoid requiring an OBS plugin unless necessary.
- Keep encoder/network backpressure isolated from capture and rendering.
- Verify frame order, timestamp behavior, disconnect/reconnect, and shutdown.

**Exit:** OBS receives a sustained AROS-produced video feed; performance and limitations are documented.

## Gate 4 — Video Toaster integration

- Define the minimal consumer-facing source contract needed by the intended AROS Video Toaster path.
- Validate multi-input switching, frame/field timing, scaling and format conversion.
- Add audio only with an explicit clock/synchronization plan.
- Assess whether Copperline/NT-Toaster code reuse is warranted and license-compatible; do not assume the emulator is the video I/O framework.

**Exit:** an AROS-side demonstration switching between at least a VHI camera/synthetic source and a network/file source, with Program output observable locally and externally.

## Cross-cutting requirements

- **Compatibility:** no breaking changes to legacy VHI ABI or consumers.
- **Portability:** no mandatory Windows, Qt, OBS, NDI or GPU dependencies.
- **Modularity:** optional components and target-specific feature profiles.
- **Memory:** bounded queues; explicit per-frame capacity and ownership; measured peak usage on m68k.
- **Timing:** monotonic timestamps, explicit conversion, defined drop/duplicate behavior.
- **Robustness:** cancellation, reconnect, source removal and cleanup are tested.
- **Scope:** do not modify AROSTCP or upstream components without a reproducible requirement.

## Suggested first engineering deliverable

A short `docs/feasibility-report.md` with direct source links, exact file/function references, build outputs, a target/protocol matrix and a recommended smallest proof of concept. Update architecture only after those findings are confirmed.
