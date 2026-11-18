# VideoBridge — architecture direction

> **Status:** conceptual architecture for design guidance, not an implementation specification, public ABI, or proof-of-concept plan. This document supersedes earlier device-first and network-first architectural sketches in this repository. Research on VHI, UVC, OBS and Exec remains valuable as adapter evidence, not as a constraint on the core design.

## Mission and boundaries

AROS VideoBridge is **generic audiovisual input/output infrastructure** for AROS. It lets applications, capture devices, files, network transports, and optional media-processing components exchange media through common contracts. The bridge is not a video editor, Video Toaster implementation, effects engine, compositor, codec suite, mandatory network server, or global multimedia daemon.

The **AROS Video Toaster** is a potential consumer and producer of VideoBridge media. Cine Toaster is a separate project. The architecture must support audio and video from the outset at the *model* level, without requiring audio processing, encoding, or synchronized rendering inside the core.

## Architectural responsibilities

```text
Applications      Capture devices      Files      Network      Optional processors
      \                 |                |           |                 /
                  Adapter / Media Node boundary
                               |
                   Media Graph (nodes/ports/links)
                               |
        +----------------------+----------------------+
        |                      |                      |
    Media Model           Media Exchange         Media Control
  sessions, tracks,     delivery semantics,     configuration,
  representations,      buffering contracts,   capabilities,
  clocks and timing     flow expectations       connection lifecycle
        +----------------------+----------------------+
                               |
                  AROS platform integration
                 (implementation-dependent)
```

These are **responsibilities**, not mandatory modules, tasks, processes, libraries, or execution stages. In particular, a graph describes connections; it does not imply a central scheduler.

### Media Graph

- **Node:** component that produces, consumes, or optionally transforms media.
- **Port:** typed interface of a node, with direction and capabilities.
- **Link:** a connection between ports with negotiated media semantics. A link is not synonymous with a socket, a transport, or a stream.
- **Graph context:** a scope for topology, visibility and coordination; whether and how contexts cross application boundaries is an implementation decision.
- Support fan-out, fan-in where an explicit component defines mixing/switching semantics, and independently configured outputs. The core does **not** perform implicit mixing, compositing, resampling or transcoding.

### Media Model

- **Session:** presentation or coordination grouping, where applicable.
- **Track:** independently represented and timed audio, video, or other media component.
- **Stream:** continuity and identity of published media; may contain one or more tracks. Neither a stream nor a track is a physical connection.
- **Media unit:** a raw frame/sample block or encoded access unit/packet with representation and timing metadata. Packetization, codec access units and decoded frames are distinct.
- **Capabilities:** describe accepted and offered representations, including codec/configuration or raw layout, timing and relevant metadata. Negotiation occurs at connection boundaries; conversions are explicit optional nodes.
- **Time:** timestamps belong to declared clock domains; relationships between tracks can be expressed without requiring a universal clock or synchronization engine.

### Media Exchange

Defines what it means to offer, accept and deliver media regardless of location. It does **not** prescribe pointer passing, Exec messages, sockets, copying, zero-copy, a central buffer pool or an always-on service. Different adapters may realize the same logical link differently.

- Ownership and lifetime must be explicit at the *contract* level; allocation strategies are not frozen here.
- Delivery has bounded-resource expectations and explicit policy choices (e.g. low-latency preview vs recording fidelity), rather than one global drop strategy.
- Compatible compressed media may pass through without decoding or re-encoding.
- A link may connect local components, components in different execution contexts, or remote peers; crossing a boundary must not silently change the meaning of the media.

### Media Control

Defines discovery-independent creation, configuration, negotiation, connection, disconnection, status and reconfiguration. Discovery may supply candidate nodes, but static wiring and application-owned graphs must work without it. Policy and mechanism remain separate: applications select priorities, reconnection behavior, permissible conversion, and latency/fidelity trade-offs.

## Architectural invariants

1. **No privileged adapter:** VHI, AHI, files, network and application interfaces are peers at the boundary; none defines the core media model.
2. **No mandatory server or scheduler:** local-only use is first-class.
3. **No implicit transforms:** a compatible representation is connected directly; otherwise negotiation fails or an explicitly selected processor participates.
4. **No forced decode:** encoded packet passthrough is valid when metadata and format constraints permit.
5. **Audio and video are modeled as tracks**, including independent clock domains; synchronization policy is not hardcoded.
6. **No unbounded exchange contract:** resource limits and backpressure semantics are observable and configurable.
7. **Control and data are distinct:** neither is tied to a particular AROS primitive.
8. **Core footprint is proportional to use:** codecs, transport protocols, discovery and complex processing are optional.
9. **Applications retain domain semantics:** editing timelines, switching, scenes, effects and recording policy are not imposed by the bridge.
10. **AROS-native, portable architecture:** respect legacy m68k constraints without encoding one CPU, driver or memory model into the public concepts.

## Architectural walkthroughs (thought experiments, not tests)

| Scenario | Required model property |
| --- | --- |
| Camera to local preview | Source and sink connect without network, daemon or mandatory codec |
| File to network output | Optional demux/packetization adapters; no mandatory raw-frame stage |
| One source to preview and recorder | Fan-out with independent delivery/latency policies |
| Video Toaster switching inputs | Toaster owns switching logic; bridge supplies independent input and Program-output ports |
| Audio + video publication | Multiple tracks and explicit clock relationships without mandatory synchronization engine |
| Application processing then republishing | Processing is an explicit node with input/output ports |
| Remote source into local application | Network is a source adapter, not a separate kind of graph |
| Incompatible formats | Negotiation reports incompatibility or requires an explicit optional converter |

## Reference comparison

| Reference | Adopt conceptually | Deliberately do not inherit |
| --- | --- | --- |
| [GPAC](https://github.com/gpac/gpac) | Composable filters, typed links, media representation boundaries | Full scheduler, extensive processing framework |
| [PipeWire](https://github.com/PipeWire/pipewire) | Node/port/link vocabulary, capability negotiation | Linux daemon and platform-specific machinery |
| [libobs](https://github.com/obsproject/obs-studio) | Sources, outputs, optional processing, asynchronous media | Scenes, rendering/compositing engine |
| [MediaMTX](https://github.com/bluenviron/mediamtx) | Publication, multi-reader distribution, track awareness | Server-centric topology and protocol-first architecture |
| [LIVE555](https://www.live555.com/liveMedia/) | Session and transport-specific timing distinctions | RTP/RTSP as universal media contracts |

These are architectural references, not dependency or code-reuse decisions. Evaluate attribution and licenses before incorporating implementation code.

## Questions to resolve architecturally (before freezing APIs)

- **Graph visibility:** what makes a node or stream discoverable outside its owning context, and who authorizes a connection?
- **Negotiation semantics:** how do formats, capabilities, clock relationships and optional transforms compose across a path?
- **Stream identity:** when do format changes, discontinuities or reconnections preserve versus replace a stream/track?
- **Fan-out semantics:** how are slow consumers isolated without imposing a global delivery policy?
- **Control model:** how do local and remote control surfaces express equivalent graph operations without conflating discovery, control and transport?
- **Extension boundaries:** which capabilities are intrinsic to the minimal bridge, and which belong in replaceable adapters?

## Implementation freedom reserved for the agent

The agent should later investigate the appropriate use of Exec libraries, devices, resources, tasks, MsgPorts, memory mechanisms, AHI/VHI and AROSTCP. **Do not infer a mandatory resident service, resource registry, shared-memory strategy, specific cancellation state machine, or public ABI from this document.** Those are engineering choices to be justified against the architecture and target constraints, not architectural premises.

The previous proof-of-concept gates in [roadmap.md](roadmap.md) and validation-oriented guidance in [media-pipeline.md](media-pipeline.md) are historical research plans; they do not override this architecture-first direction.
