# Architecture: sources, frames, consumers, sinks

## Central distinction

**Capture** and **publication** are separate concerns. The existing AROS VHI contract primarily describes a capture device and digitized image. OBS interoperability can be achieved either by publishing a stream consumed by OBS or by implementing a host-side OBS source plugin; neither requires OBS to be embedded in AROS.

```text
  UVC / VHI driver ----+
  network decoder -----+--> source adapters --> bounded frame exchange --> video consumers
  file decoder --------+                                 |                 (Toaster, preview, etc.)
  synthetic source ----+                                 |
                                                         +--> display sink
                                                         +--> encoder / file sink
                                                         +--> network publisher --> OBS
```

This is a **candidate** design, not an existing AROS implementation.

## Preserve the VHI contract

AROS `compiler/include/vhi/vhi.h` defines `VHI_METHOD_OPEN/CLOSE/SET/GET/PERFORM`, `VHI_DIGITIZE_PICTURE`, and `struct vhi_image` with RGB24, YUV and JPEG representations. `struct vhi_digitize` supports caller-provided allocation/free callbacks. The public `vhi_api` entry point is specified in `workbench/libs/vhi/vhi.conf`.

The legacy `vhi_image` has no explicit presentation timestamp, duration, sequence number, audio stream, or discontinuity indicator. Do **not** change its layout in place. A virtual VHI source can initially implement the existing pull-style digitize operation over a buffered decoded stream. A separate optional frame envelope may carry metadata for clients that opt in.

`VHI_DC_TYPE_TCPIP` is a historical camera classification, not evidence of a working generic network-stream implementation.

## UVC is a lower-layer capture implementation

`rom/usb/classes/video/uvc.class.c` and `compiler/include/devices/uvc_capture.h` expose USB video capture and mode negotiation, including YUY2, MJPEG and grayscale descriptors. The documented `UVCM_GRAB` interface supplies RGB24 buffers and returns bytes or zero on failure. The header also declares optional VHI UVC GET/SET extensions for mode, device information and frame interval.

Investigate the actual UVC-to-VHI driver and buffer ownership before assuming zero-copy or asynchronous capture. Do not confuse USB's internal streaming with an application-level push API.

## Candidate frame envelope (not an ABI proposal)

A versioned, size-tagged metadata record could contain:

- source identifier and generation
- pixel format, width/height, stride/plane layout
- monotonic presentation timestamp and optional duration
- frame sequence and discontinuity flags
- buffer ownership/release mechanism and explicit capacity

Keep audio on a separately synchronized path until a proven use case requires a shared AV envelope. Specify clock conversion, wraparound, lifetime, and cancellation before stabilizing any ABI.

## Concurrency and memory

- Source workers perform potentially blocking device, file, or network I/O.
- Consumers never hold source locks while encoding or sending over the network.
- Bounded queues with an explicit drop policy prevent unbounded latency and memory growth.
- Disconnection, timeout, end-of-stream, format change, and shutdown must be first-class states.
- Use Exec tasks/MsgPorts where appropriate; evaluate ownership and task affinity before passing frame pointers.
- Preserve low-memory m68k operation with configurable queue depth and optional transports/codecs.

At 640x480 RGB24, a single frame is 921,600 bytes; eight frames consume about 7.0 MiB before decoder and network buffers. Measure actual target memory, throughput and copies rather than assuming desktop-class resources.

## Interoperability options

### OBS as output

1. **Network publication (initial candidate):** encode and publish a format/protocol that OBS can ingest. Choose the protocol only after validating the actual AROS FFmpeg/network stack and OBS ingest behavior.
2. **OBS source plugin (later):** a plugin on the OBS host consumes an AROS frame transport. This may avoid on-device encoding, but requires a defined wire protocol and a maintained plugin.
3. **Open-protocol alternatives:** investigate interoperable RTP/RTCP, RIST, and other open transport implementations where supported by measured target capabilities. NDI is proprietary and is not an implementation candidate; its presence in NT-Toaster is retained only as upstream historical context.

### OBS as input

OBS output (e.g. a supported network feed) can become a source for a network decoder/virtual VHI adapter. This is a separate direction from publishing AROS Program output to OBS.

## Architectural decision gates

Do not introduce a new library or public API until a prototype proves that the legacy VHI API cannot express the required operation adequately. Keep internal source/sink interfaces private during initial experiments. Avoid modifying the existing AROS TCP/IP stack unless a reproducible missing network primitive is demonstrated.

See [research](research.md) for upstream evidence and [roadmap](roadmap.md) for validation.

## Composable media pipeline

The source/frame/sink sketch is a high-level view, not a requirement for monolithic adapters. Separate source acquisition, demuxing, decoding, pixel conversion, processing, encoding, muxing, transport, and output. Keep each optional and negotiate capabilities before connecting modules. Where source and destination formats permit, compressed packet passthrough should avoid decode/re-encode. A media editor or timeline system may orchestrate seeks and clip ranges but should not become part of the core. See [media-pipeline.md](media-pipeline.md).

## Architecture review against reference projects (November 2026)

**Status: candidate logical architecture; not an ABI, daemon, build plan or runtime validation.** This review corrects an earlier device-first / proof-of-concept emphasis. The goal at this stage is to design a generic AROS audiovisual I/O architecture. Network, files, applications, VHI and AHI are interchangeable adapter domains, not architectural centers.

### Reference comparison

| Reference | Architectural lesson | Gap in the previous Endpoint / Stream Fabric / AROS Services model |
| --- | --- | --- |
| [MediaMTX](https://github.com/bluenviron/mediamtx) | Published streams, multiple readers, media tracks, protocol adapters and bounded distribution | Stream naming, tracks and publication lifecycle were underspecified; do not turn the network server into a mandatory core |
| [GPAC](https://github.com/gpac/gpac) | Filters with input/output connections, packet metadata, capability matching, composable transforms | Connections, packet-versus-frame semantics and optional transform nodes were missing |
| [libobs](https://github.com/obsproject/obs-studio) | Separation of sources, outputs and encoders; asynchronous media and timestamps | Bidirectional application integration and async delivery contracts were unclear; composition/scenes remain outside scope |
| [LIVE555](https://www.live555.com/liveMedia/) | Session/subsession, transport-specific timing and lifecycle | Presentation/session and track semantics were not modeled explicitly; avoid coupling the core to RTP/RTSP |
| [PipeWire](https://github.com/PipeWire/pipewire) | Nodes, ports, links, capabilities and buffer exchange | Endpoint conflated an entity with its connections; shared-buffer negotiation and graph topology were underspecified |

This is an architectural comparison, not a claim that the reference implementations can be embedded in AROS or share compatible licenses.

### Revised logical architecture

```text
 Network adapters       Application adapters       File / device adapters
          \                    |                       /
                Media Nodes and typed I/O Ports
                              |
                  Media Graph / Stream Fabric
              Links, routing, publication, subscription
                         /           \
               Media Model          Buffer Model
         sessions/tracks/formats   ownership/queues/flow
                 clocks/timing     lifetime/copy boundaries
                         \           /
                    Control and Lifecycle
           discovery hooks, negotiation, state, recovery
                              |
                    AROS Platform Integration
            Exec tasks/ports/memory; AROSTCP as adapter
```

These are **logical responsibilities**, not necessarily separate libraries, processes, threads, services or allocation domains. The graph is a *connectivity model*, not an obligatory GPAC-like processing scheduler. Local links must not be forced through a network server.

### Conceptual contracts (names provisional)

- **MediaNode:** a source, sink or optional transformer; a transformer is a node with both input and output ports. The core routes media but does not implement effects, compositing, codecs or editing.
- **MediaPort:** a typed input/output interface with capabilities, accepted media representations and lifecycle.
- **MediaLink:** a negotiated connection between compatible ports, distinct from the stream being carried. Support fan-out and independent consumer policies.
- **MediaStream:** publishable identity and continuity of media; not synonymous with a socket or link.
- **MediaSession / MediaTrack:** presentation-level grouping and independently timed video/audio/other tracks; define synchronization relationships without assuming all tracks share one clock.
- **MediaUnit:** either an encoded packet or a raw frame, with explicit representation, timestamp domain, duration, sequence/discontinuity and format/configuration changes. Do not silently treat RTP packets, codec access units and decoded frames as interchangeable.
- **MediaBuffer:** payload storage, immutable while shared, with capacity, owner, reference/release contract and optional copy/transfer semantics. Pointer sharing is not automatically portable zero-copy.
- **MediaCapabilities / negotiation:** compatible format selection across **both** ports and their link. If a transform is needed, represent an explicit optional node rather than hidden conversion in the router.
- **Control / lifecycle:** create, publish, connect, subscribe, renegotiate, disconnect, cancel, error, recovery and end-of-stream; control-plane messages and payload delivery have separate contracts.

### Policies versus mechanisms

The fabric provides routing, subscriptions, lifecycle and bounded flow-control mechanisms. Applications/adapters choose policies: live-preview frame drop versus loss-intolerant recording, queue limits, reconnect strategy, preferred codecs, and whether to permit format conversion. Discovery (including optional DNS-SD) supplies candidates but must not be required for routing or static configuration.

### AROS-specific architectural decisions still open

1. **Execution and deployment:** compare library-only, Exec service and hybrid models; a library API alone does not imply cross-process shared state, and a mandatory resident daemon is not justified yet.
2. **Ownership and memory:** establish how tasks exchange buffer references, allocation constraints for m68k and other targets, optional copy paths, DMA/cache restrictions and lifetime on cancellation.
3. **Clock model:** define monotonic/timebase conversion, track alignment, timestamp discontinuities and wraparound without mandating a global real-time scheduler.
4. **Capability and graph lifecycle:** define who authorizes links, handles renegotiation and recovers from endpoint loss; avoid global locks held across blocking I/O.
5. **Module boundaries:** keep transport, codecs, demux/mux, devices and optional discovery independent of the minimal core; respect target-specific build footprints.
6. **Security and trust:** local-versus-remote endpoint permissions, network authentication and resource exhaustion limits belong at explicit boundaries, not in an implicit trusted graph.

### Scope and architectural acceptance criteria

A coherent design must represent: (a) network-to-network compressed passthrough without mandatory decode/re-encode; (b) one producer and multiple consumers with independent flow policy; (c) an application consuming and republishing a transformed stream; (d) a session with independently timed audio and video tracks; and (e) local-only operation without network or a mandatory daemon. These are **design scenarios**, not an implementation or testing schedule.

**Do not prematurely freeze:** public ABI, service placement, wire protocol, scheduling model, allocator, codec, VHI integration details, or a mandatory process topology. First reconcile the conceptual contracts with actual AROS library/device/resource/task conventions. Keep original code provenance and review licenses before reusing third-party implementation code.
