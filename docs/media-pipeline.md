# Composable media pipeline: architectural research

**Status:** proposed design boundaries, not an implemented AROS ABI or a commitment to new libraries.

## Purpose and concrete use cases

AROS VideoBridge is reusable audiovisual input/output infrastructure. Two motivating examples anchor the research:

- **AROS Video Toaster:** consume VHI/UVC, file or network media and publish clean Program output without coupling the switcher to network protocols.
- **OBS interoperability:** publish AROS-generated video through a compatible network stream or an explicitly implemented host-side receiver/OBS source adapter.

Other potential clients include camera preview/recording, media playback, network-camera monitoring, and editing applications. A timeline system such as OpenTimelineIO may direct clip ranges and seeks; timeline semantics and editing decisions remain outside the VideoBridge core. AROS Prism display topology and desktop extension are separate concerns.

## Pipeline boundaries

```text
Device / file / network
        |
Source acquisition / protocol adapter
        |
Optional demux -> encoded packets ----+----> compatible packet passthrough
        |                              |
Optional decoder                      |
        |                              |
Raw frames -> optional conversion / processing
        |                              |
Optional encoder <--------------------+
        |
Optional mux / packetization
        |
File / local consumer / network transport / host adapter
```

These stages are logical responsibilities, not mandatory separate processes, shared libraries, or C interfaces. Some drivers provide already-decoded frames; some network sources carry compressed packets; some consumers require raw frames. Avoid decode/re-encode where a compatible encoded packet path exists, subject to container, timestamp, codec-configuration and receiver requirements.

## Shared contracts to investigate

- **Media capabilities:** compressed codec and profile, raw pixel format, dimensions, stride/plane layout, interlacing/field order, frame cadence and color metadata where needed.
- **Timing:** explicit clock domains, timestamps, duration, discontinuities, seek and end-of-stream behavior; audio requires separate synchronized handling.
- **Buffer lifetime:** bounded ownership/release rules, no unbounded allocation or hidden frame copies, explicit backpressure/drop policies.
- **Negotiation:** source and destination capabilities determine whether demux, decode, conversion, encode, mux or transport adaptation is necessary.
- **Error and recovery:** format changes, cancellation, stalled sources, slow sinks, disconnect/reconnect and partial packets.
- **Module discovery:** optional codecs, protocol adapters and media backends must not impose dependencies on applications that do not use them.

A frame is not the same as an encoded packet. A codec is not a transport; a muxer/container is not a codec; an application-level source/sink is not necessarily a socket endpoint.

## Native AROS integration

VHI remains a capture compatibility interface, not a general streaming API. Do not change the public VHI ABI to carry timestamps or network protocols. Investigate existing UVC/VHI capture, AHI for audio, FFmpeg port capabilities, AROSTCP sockets and Exec task/MsgPort semantics before inventing replacements. Preserve original source attribution and evaluate third-party licenses independently.

## Legacy m68k profiles

Keep heavyweight codecs and transports optional. For 640x480 RGB24, one frame is 921,600 bytes; eight frames consume about 7 MiB before other buffers. At 30 fps the uncompressed stream requires roughly 221 Mbit/s, above 100 Mbit Ethernet before overhead. A host-side encoder removes on-AROS encode cost but does not solve raw bandwidth. Investigate compressed passthrough, reduced resolution, small queues, explicit drops and target-specific build profiles.

## Licensing and protocol selection

Favor open implementations and standards with licenses suitable for AROS. NDI is proprietary and **not a proposed VideoBridge implementation target**. References to NDI in NT-Toaster research describe upstream functionality only. Evaluate codecs separately for patent and licensing constraints; an open transport does not guarantee that every codec used over it is unrestricted.

## Validation before public APIs

1. Verify one VHI/UVC source and its buffer lifetime on a concrete AROS target.
2. Verify one file/network source and distinguish packet passthrough from decoded-frame delivery.
3. Demonstrate a local frame consumer with bounded memory and predictable cancellation.
4. Demonstrate an end-to-end Video Toaster Program or synthetic frame output to an OBS-compatible receiver.
5. Measure copies, CPU, RAM, latency and bandwidth on legacy m68k and at least one less-constrained target.
6. Only then decide whether common contracts justify a public library/ABI.

See [architecture.md](architecture.md), [streaming-protocols.md](streaming-protocols.md), and [toaster-program-output.md](toaster-program-output.md) for existing evidence and alternatives.
