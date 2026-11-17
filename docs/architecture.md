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
