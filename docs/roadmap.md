# Architecture-first roadmap

> **Current phase: architectural design, not implementation or proof of concept.** This roadmap supersedes the earlier VHI/UVC-first experimental gates. No code, AROS ABI, task model, transport protocol or prototype is authorized by this document.

## North star: heterogeneous input, application-owned processing, independent output

The [reference scenario](reference-scenario-video-toaster-obs.md) is:

```text
Network stream -- [network adapter] --+
                                       |
Local file ----- [file adapter] -------+--> AROS Video Toaster --> Program
                                       |                           |
Capture device - [capture adapter] ----+                  [output adapter]
                                                                   |
                                                               OBS Studio
```

Optional decoding, conversion, encoding, packetization and other processing are **explicit replaceable nodes**, never mandatory functions of the core. The Video Toaster owns switching and composition. OBS is a reference consumer, not a core dependency. Audio tracks are included in the model even when a particular scenario uses video only.

A proposed architecture is inadequate if it can express this scenario only by embedding network/file/capture-specific behavior in the Video Toaster or by mandating a central server.

## Decision A — Minimal architectural kernel

**Question:** What must every independent audiovisual component understand to connect to another?

- Stabilize conceptual responsibilities for **nodes, typed ports, links, media representation and capabilities**.
- Distinguish stream/track identity from links and physical transport.
- Determine which concepts are essential for a trivial local camera-to-preview path and which are optional for richer scenarios.
- Separate architectural contracts from adapter capabilities, graph planners and processing engines.
- Avoid promoting `GraphContext`, `Session` or any other conceptual term into mandatory runtime objects without a clear reason.

**Exit:** a small, internally consistent vocabulary and a core-versus-extension responsibility matrix.

## Decision B — Connection and capability negotiation

**Question:** What makes two ports compatible?

- Specify compatibility semantics for raw video, encoded video, raw audio, encoded audio and extensible media.
- Keep packetization, codec access units and decoded frames distinct.
- Express relevant format, timing and delivery constraints without imposing a universal format.
- Prefer direct compatible links; incompatible ports must report a reason.
- Permit explicitly chosen optional converter/decoder/encoder nodes, but **do not require** automatic graph planning or hidden conversion.

**Exit:** a conceptual connection contract that covers direct passthrough, incompatible formats, and explicitly composed conversion paths.

Reference: [connection model](connection-model.md).

## Decision C — Application-facing input/output contracts

**Question:** How can the Toaster receive three unrelated inputs and publish Program without knowing their adapters?

- Describe how applications expose accepted input ports and offered output ports.
- Keep application-specific behavior (switching, effects, composition, editing, recording policy) outside VideoBridge.
- Treat a Program output as publishable media independent of its downstream consumer.
- Define how one Program can serve OBS, local preview and recording, with independent destination policies.
- Clarify the relationship among source identity, stream continuity and application-owned Program identity.

**Exit:** an application contract and diagrams for the three-input/OBS scenario and its multi-output extension.

## Decision D — Media timing, continuity and change

**Question:** What media semantics survive changes in source, representation or connection?

- Define track-level timing and declared clock domains, including independently timed audio/video.
- Distinguish source interruption, link interruption, format change and stream replacement.
- State what information a consumer needs to react to discontinuity or renegotiation.
- Avoid assuming that VideoBridge itself performs synchronization, buffering, resampling or frame-rate conversion.

**Exit:** a semantic model for identity, timing and reconfiguration, not an Exec cancellation state machine.

## Decision E — Distribution, placement and extensibility

**Question:** How does the same architecture serve local applications and optional network publication?

- Keep local links first-class, with no required daemon, central registry or scheduler.
- Separate control/discovery from media exchange; discovery remains optional.
- Make fan-out and independent consumer policies expressible without forcing one queueing implementation.
- Place VHI/UVC, AHI, file I/O, AROSTCP/protocols, OBS interoperability and codecs behind replaceable boundaries.
- Preserve low-footprint m68k viability by requiring no unused adapter or heavyweight media engine in the minimal design.

**Exit:** a deployment-neutral conceptual architecture and clear extension boundaries.

## Architecture review: apply the same questions to every proposal

| Reference case | Essential property |
| --- | --- |
| Capture → preview | Local direct connection, no network or codec dependency |
| Encoded camera → recorder | Compatible encoded passthrough |
| Encoded source → raw-only sink | Explicit incompatibility or optional decoder |
| Stream + file + capture → Video Toaster → OBS | Heterogeneous inputs and independent Program publication |
| Program → OBS + preview + recorder | Fan-out with independent destination expectations |
| Audio + video tracks | Declared timing relationships, no mandatory mixer/sync engine |
| Source reconnects or changes format | Identity and continuity semantics remain clear |

## Design principles for the agent

1. **Architecture before implementation.** Resolve conceptual responsibilities and their boundaries first.
2. **Interoperability is the goal, not feature accumulation.** VideoBridge is more than a router because components share media and connection contracts, not because the core performs more processing.
3. **No privileged source type.** File, network, capture and application media are peers.
4. **No hidden work.** No implicit decoding, transcoding, compositing or format conversion.
5. **No mandatory infrastructure.** A local-only path must not require network, daemon, central scheduler or discovery.
6. **No premature ABI decisions.** Exec tasks, MsgPorts, libraries, devices, resources, buffers and synchronization are later engineering decisions.
7. **Preserve AROS/m68k viability.** The minimal architecture should not require desktop-class runtime components.
8. **Preserve provenance.** Existing research and source attributions remain valid; reuse of code requires separate license review.

## Existing research: informative, not a gate sequence

- [Architecture](architecture.md): mission, responsibilities and invariants.
- [Connection model](connection-model.md): compatibility and composition semantics.
- [Video Toaster → OBS reference scenario](reference-scenario-video-toaster-obs.md): end-to-end architectural example.
- [Media pipeline research](media-pipeline.md): codecs, demux, formats, buffers and platform considerations.
- [Streaming protocols](streaming-protocols.md): candidate network adapters, not a mandatory protocol choice.
- [Research references](research.md): upstream evidence and provenance.

The previous feasibility/prototype plan (virtual VHI, file/network ingest, OBS demo, Video Toaster integration) may inform a **later implementation plan** once these decisions are stable. It is not the current roadmap.

## Next documentation deliverables

- A concise **core-versus-extensions responsibility matrix** (Decision A).
- A **port compatibility and negotiation contract** illustrated by direct and converted paths (Decision B).
- An **application-facing Program/input contract** using the three-input Toaster scenario (Decision C).

No proof of concept or implementation milestone is required to complete these deliverables.
