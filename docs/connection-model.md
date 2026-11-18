# Connection semantics: the architectural distinction from a router

> **Status:** architectural design direction. No public API, implementation mechanism, runtime scheduler or prototype is specified here. Read with [architecture.md](architecture.md).

## Thesis

A router forwards media between known endpoints. **VideoBridge connects independently developed audiovisual components by a common model of ports, media representations, capabilities and connection policy.** Routing remains the minimal operation; interoperability and composition are the added value. Do not add a feature to the core merely because a full media framework offers it.

## What a connection means

A **port** advertises what it can produce or consume. A **link** represents an agreement between two ports about the representation being exchanged and relevant delivery expectations. A **stream** is the identity/continuity of the media, independent of any one link. A **track** identifies an individual media component (video, audio, etc.). A **node** owns application-specific behavior, whether capture, playback, transform, switch, encode, file I/O or network I/O.

The graph expresses *which* ports connect. The link expresses *what* is exchanged and under which constraints. The nodes determine *how* media is produced or consumed. The core must not secretly implement node behavior.

## Connection resolution

A connection request is resolved conceptually as follows:

1. **Discover or explicitly identify ports.** Discovery is optional; direct references must be sufficient.
2. **Compare capabilities.** Include media type, encoded vs raw representation, required format/configuration, relevant timing properties and delivery constraints. A mere common media type (e.g. `video`) is not sufficient.
3. **Choose a compatible representation.** The negotiated result is observable to both sides; a connection cannot silently reinterpret units.
4. **If incompatible, report why.** An application or an optional graph-building component may select explicit converters/decoders/encoders and retry; the core does not insert them invisibly.
5. **Establish a link.** Its logical semantics are independent of local memory, inter-task exchange, file or network realization.
6. **Handle change.** Reconfiguration, format change and discontinuity are explicit events or negotiated transitions; do not silently substitute incompatible media.

This is a semantic model, **not** a required call sequence, state machine or mandatory runtime service.

## Capability model: what must be expressible

| Dimension | Architectural requirement |
| --- | --- |
| Media kind | Audio, video and extensible additional types |
| Representation | Distinguish raw media, encoded access units, and transport packets |
| Format | Codec/profile/configuration or raw sample/pixel format and layout |
| Geometry / rate | Where relevant: dimensions, sample rate, channels, cadence |
| Time | Declared timestamp domain and relationship to other tracks |
| Delivery | Whether consumer requires continuity, low latency, lossless delivery or accepts drops |
| Configuration change | Whether a consumer can accept changes without rebuilding a link |

Do not turn every optional property into a required field for every port. Capabilities should be extensible and allow minimal descriptions when enough to decide compatibility.

## Concrete architectural examples

### 1. Camera to preview

```text
VHI/UVC adapter [raw video] --> Preview [compatible raw video]
```

Direct connection where representations agree. No network, codec, server or generic processing stage is required.

### 2. Encoded camera to recorder

```text
Camera [MJPEG access units] --> Recorder [accepts MJPEG access units]
```

No decode/re-encode requirement. The recorder remains responsible for any file container adaptation.

### 3. Encoded camera to raw-only consumer

```text
Camera [MJPEG] --X--> Consumer [RGB24 only]

Camera [MJPEG] --> Optional decoder [MJPEG -> RGB24] --> Consumer [RGB24]
```

The initial request is incompatible. A decoder is an explicit node, chosen by application policy or optional graph construction. No automatic hidden conversion.

### 4. Video Toaster

```text
Camera ------> [input port]          [Program output] --> Preview
File source -> [input port] Video Toaster                --> Recorder
Network ----> [input port]          [Program output] --> Network adapter
```

The Video Toaster owns switching/composition semantics; VideoBridge exposes independent input/output connections. The diagram is logical: multiple outputs may be represented by links or separate ports depending on their capabilities.

### 5. Audio and video

```text
Video track -----+
                 +--> Publisher / consumer with explicit track timing
Audio track -----+
```

Track grouping and clock relationships are model-level concepts. The core does not become a mixer or synchronization engine.

### 6. One source, heterogeneous consumers

```text
Source [encoded video] --+--> Recorder [encoded video]
                         +--> Decoder --> Preview [raw video]
```

Fan-out does not imply one format for all consumers, nor does a slow consumer dictate every link's delivery policy.

## Minimal core versus extensions

**Core architectural obligations:** graph vocabulary (nodes/ports/links), media/track identity, capability compatibility, explicit connection semantics, timing metadata semantics, delivery policy contracts and lifecycle visibility.

**Replaceable participants:** capture, audio hardware, file access, protocol adapters, demuxers, decoders, encoders, converters, compositors, switchers, recorders, renderers, discovery providers and optional graph planners.

**Not required:** a centralized server, persistent global registry, automatic graph optimizer, universal scheduler, built-in codec, timeline editor, effects engine, mandatory transcoding or always-on background task.

## Design choices and non-goals

- **Negotiation should be simple before it is clever.** Compatible direct connections are the baseline; graph search and automated insertion of transforms are optional orchestration features.
- **No single universal media payload.** Raw video, audio samples, encoded access units and transport packets carry different semantics. A common envelope must not erase them.
- **Avoid an overly rich mandatory taxonomy.** Session, stream and track are useful where needed; a simple local camera-preview link should not require constructing a full presentation/session hierarchy.
- **Preserve independent fan-out policy.** Preview can prioritize freshness while recording can prioritize completeness, subject to source limitations.
- **Treat local and remote as placement choices**, not different media abstractions. This does not imply equal cost or capabilities.
- **Don't conflate interoperability with format conversion.** Interoperability means components can declare and resolve compatibility; conversion is one optional way to bridge incompatibility.

## Architectural acceptance questions for future agent decisions

Before adding a core feature, ask:

1. Is it needed for *any* two independent media components to connect, or only for a particular adapter/application?
2. Does it preserve direct pass-through when both sides already agree?
3. Does it avoid mandatory network, daemon, decoding, transcoding and central scheduling?
4. Can it describe audio and video without forcing a shared clock or processing pipeline?
5. Does it permit optional richer capabilities without increasing the minimal local case?
6. Can a component be replaced without requiring all its peers to learn a proprietary interface?

If the answer is no, prefer an adapter, optional node or application-level orchestration rather than expanding the VideoBridge core.

## Open architecture questions

- What is the smallest extensible vocabulary for capabilities without turning negotiation into a full constraint solver?
- Which continuity rules distinguish stream identity from individual link lifetime?
- How are capability changes communicated without requiring a mandatory control daemon?
- Should an optional graph planner be standardized, or is a common port/link contract sufficient?
- Which media metadata is universally necessary versus representation-specific?

Resolve these as architecture questions. Buffer allocation, Exec message layout, thread supervision, cancellation primitives and ABI design belong to later engineering work.
