# Reference architecture scenario: three inputs, Video Toaster, OBS output

> **Status:** illustrative end-to-end architecture, not a commitment to a specific streaming protocol, decoder, encoder, hardware target or implementation sequence.

## Purpose

Use one coherent example to test whether AROS VideoBridge is truly a **generic audiovisual interoperability infrastructure**, rather than a network router or a capture-device wrapper.

Three heterogeneous media sources feed **AROS Video Toaster**:

1. **Network stream** — supplied by a network/protocol adapter, with optional depacketization and decoding when the Toaster requires raw frames.
2. **Local media file** — supplied by a file/demux adapter, with optional decoding as needed.
3. **Capture device** — supplied by a capture adapter, e.g. VHI/UVC, with the device's actual supported representation.

The Toaster selects/switches/composes its inputs and produces one **Program output**. An output chain then publishes that Program as a stream consumed by **OBS Studio**.

```text
  Network stream --- [network input adapter] -- [optional decoder] --+
                                                                     |
  Local file ------ [file input adapter] ---- [optional decoder] ----+--> [AROS Video Toaster]
                                                                     |     inputs / switch / compose
  Capture device -- [capture adapter] ----- [optional conversion] --+               |
                                                                                 Program
                                                                                   |
                                                                      [output media port]
                                                                                   |
                                                                       [optional encoder]
                                                                                   |
                                                                     [streaming output adapter]
                                                                                   |
                                                                             OBS Studio
```

All adapters, converters and codecs are **replaceable nodes**, not required components of the VideoBridge core. The Toaster is the application node responsible for switching and composition. The output adapter chooses an OBS-compatible ingest path based on the eventual target constraints; do not prescribe RTMP, RTSP, RTP, MPEG-TS or another protocol at the architectural level.

## Architectural contracts demonstrated

| Concern | What this scenario proves conceptually |
| --- | --- |
| Source neutrality | Stream, file and capture all expose compatible media ports to the application |
| Representation negotiation | Each input negotiates with the Toaster's actual accepted formats; optional decoding/conversion is explicit |
| Application ownership | Switching, effects, composition and Program semantics belong to the Toaster |
| Output independence | The Toaster publishes Program without knowing whether the destination is OBS, a file or a local preview |
| Codec independence | Encoding is an optional participant between Program and the streaming output |
| Network neutrality | The network is an adapter on either side of the graph, not the architecture itself |
| Time model | Each source can have its own timestamps/clock; the Toaster decides how to align sources for Program |
| Media extensibility | Audio can be represented by independent tracks and negotiated separately, without forcing audio processing into the core |

## Architectural distinction: router versus bridge

A router could select and forward one already-compatible source to one output. This scenario requires more: *independent source adapters, a common media/port contract, explicit compatibility negotiation, optional transform nodes, an application that consumes and produces media, and an independent publication path*. The VideoBridge enables composition of these participants; it does not implement their specialized behavior.

## Questions for later architectural refinement

- What minimum port capabilities must the Toaster declare to accept each input?
- Does Program represent one video track or a session with optional audio tracks?
- How should an application express that a source changes format or becomes temporarily unavailable?
- What output properties are selected by the Toaster versus the streaming publisher?
- Can the same Program feed OBS, a local preview and a recorder simultaneously without changing the Toaster?

These are design questions. Do not treat the example as a mandate to build a proof of concept, choose a protocol or implement a specific Exec task/message architecture.
