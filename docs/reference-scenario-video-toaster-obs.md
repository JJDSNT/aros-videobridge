# Reference architecture scenario: three inputs, Video Toaster, OBS streaming and file recording

> **Status:** illustrative end-to-end architecture, not a commitment to a specific streaming protocol, decoder, encoder, hardware target or implementation sequence.

## Purpose

Use one coherent example to test whether AROS VideoBridge is truly a **generic audiovisual interoperability infrastructure**, rather than a network router or a capture-device wrapper.

Three heterogeneous media sources are acquired **through VideoBridge source adapters** and feed **AROS Video Toaster**:

1. **Network stream** — supplied by a network/protocol adapter, with optional depacketization and decoding when the Toaster requires raw frames.
2. **Local media file** — supplied by a file/demux adapter, with optional decoding as needed.
3. **Capture device** — supplied by a capture adapter, e.g. VHI/UVC, with the device's actual supported representation.

The Toaster selects/switches/composes its inputs and produces one **Program output**. The **same Program output** feeds two simultaneous, independent destinations: **a network stream consumed by OBS Studio** and **a locally recorded media file**. Neither output is a prerequisite for the other.

```text
  Network stream ---- [network adapter] ----+
                                             |
  Local file -------- [file adapter] -------+--> [VideoBridge: acquisition]
                                             |     (ports, capabilities, links)
  Capture device ---- [capture adapter] ----+                |
                                               [explicit optional transforms]
                                                            |
                                                    [AROS Video Toaster]
                                                   switch / compose / Program
                                                            |
                                               [VideoBridge: distribution]
                                               (fan-out, negotiated links)
                                                     /              \
                                        [optional encoder]     [optional encoder]
                                               |                     |
                                        [stream adapter]       [file writer/muxer]
                                               |                     |
                                           OBS Studio            Recorded file
```

**Acquisition and distribution are two roles of the same VideoBridge architecture, not necessarily two separate libraries, services or instances.** The Toaster interacts with media ports on both sides; it is not responsible for the concrete source and destination adapters.\n\nAll adapters, converters and codecs are **replaceable nodes**, not required components of the VideoBridge core. The Toaster is the application node responsible for switching and composition. The streaming adapter chooses an OBS-compatible ingest path based on the eventual target constraints; do not prescribe RTMP, RTSP, RTP, MPEG-TS or another protocol at the architectural level.

## Architectural contracts demonstrated

| Concern | What this scenario proves conceptually |
| --- | --- |
| Acquisition via VideoBridge | Stream, file and capture are exposed to Toaster through the same port/link/capability model |
| Representation negotiation | Each input negotiates with the Toaster's actual accepted formats; optional decoding/conversion is explicit |
| Application ownership | Switching, effects, composition and Program semantics belong to the Toaster |
| Distribution via VideoBridge | The same Program feeds OBS streaming and local file recording through independent links, without changing Toaster logic |
| Codec independence | Each output can negotiate its own representation and optional encoder; sharing an encoder is an optimization, not a requirement |
| Network neutrality | The network is an adapter on either side of the graph, not the architecture itself |
| Time model | Each source can have its own timestamps/clock; the Toaster decides how to align sources for Program |
| Media extensibility | Audio can be represented by independent tracks and negotiated separately, without forcing audio processing into the core |

## Architectural distinction: router versus bridge

A router could select and forward one already-compatible source to one output. This scenario requires more: *independent source adapters, a common media/port contract, explicit compatibility negotiation, optional transform nodes, an application that consumes and produces media, and symmetric acquisition/distribution contracts plus two independent output paths (network publication and file recording)*. The VideoBridge enables composition of these participants; it does not implement their specialized behavior.

## Questions for later architectural refinement

- What minimum port capabilities must the Toaster declare to accept each input?
- Does Program represent one video track or a session with optional audio tracks?
- How should an application express that a source changes format or becomes temporarily unavailable?
- What output properties are selected by the Toaster versus the streaming publisher?
- How do the simultaneous OBS stream and file recorder express independent format, continuity, latency and failure policies?\n- Can local preview be added as a third consumer without changing the Toaster?

These are design questions. Do not treat the example as a mandate to build a proof of concept, choose a protocol or implement a specific Exec task/message architecture.
