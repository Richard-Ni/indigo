# Frame View RGB24 Streaming Interface

## Purpose

This interface provides a low-latency Frame View preview path for cameras that already produce RGB24 pixels. It is modeled after the Panel View dedicated WebSocket path, but it has no panel geometry, centroid coordinates, star detection, HFD, FWHM, or image upscaling.

The channel is global and carries exactly one stream. The header has no device identifier, and every publisher broadcasts to every connected client, so two cameras streaming at once would interleave frames indistinguishably on the same socket. This is an accepted constraint: the supported deployment streams from one camera at a time.

The Imager Agent remains the control plane. It accepts and proxies `CCD_STREAMING`; it is not a binary pixel relay. RGB24 frames are published by the selected CCD driver through libindigo and are sent to browser clients on a dedicated binary WebSocket.

## Endpoints

Control WebSocket:

```text
ws://<host>:7624/
```

Use the normal INDIGO JSON protocol to select the camera and set:

```json
{
  "newNumberVector": {
    "device": "Imager Agent",
    "name": "CCD_STREAMING",
    "items": [
      { "name": "COUNT", "value": -1 },
      { "name": "EXPOSURE", "value": 0.1 }
    ]
  }
}
```

RGB24 preview WebSocket:

```text
ws://<host>:7624/api/streaming/frame
wss://<host>:7624/api/streaming/frame
```

The frame socket is server-to-client only. A client does not send frame requests after connecting. Close it when `CCD_STREAMING` leaves `Busy` or when the page unloads.

## Wire Format

One binary WebSocket message contains exactly one frame:

```text
[24-byte little-endian header][width * height * 3 RGB bytes]
```

| Offset | Size | Type | Field | Value |
|---:|---:|---|---|---|
| 0 | 4 | u32 | `magic` | `0x31565246`, bytes `FRV1` |
| 4 | 2 | u16 | `width` | frame width |
| 6 | 2 | u16 | `height` | frame height |
| 8 | 4 | u32 | `seq` | server-wide published frame counter |
| 12 | 8 | u64 | `timestamp_ms` | server realtime epoch milliseconds |
| 20 | 2 | u16 | `bpp` | always `24` |
| 22 | 1 | u8 | `channels` | always `3` |
| 23 | 1 | u8 | `reserved` | always `0` |
| 24 | `width*height*3` | u8[] | `pixels` | row-major, top-down, interleaved `R,G,B` |

The browser must reject a message unless all of these hold:

```text
magic == 0x31565246
bpp == 24
channels == 3
width > 0 && height > 0
message_length == 24 + width * height * 3
```

`width` and `height` are `uint16_t`. The libindigo publisher also rejects a payload whose computed byte count overflows `uint32_t` or does not exactly equal `width * height * 3`.

The header is written as a packed struct in host byte order, so the little-endian layout above holds on little-endian hosts only. INDIGO targets x86 and ARM, both little-endian; big-endian servers are out of scope for this interface.

`seq` counts frames accepted by the publisher process-wide. It is not reset when a client connects, and it advances even while no client is attached, so the first `seq` a client observes is arbitrary. Gaps are normal and expected: this path is deliberately drop-tolerant, and a client that cannot keep up simply misses frames. `seq` is the only signal a client has for detecting those drops; it must never be used to index or reassemble anything.

## CCD Driver Integration

Include:

```c
#include <indigo/indigo_server_tcp.h>
```

`indigo_frame_preview_publish_rgb24()` and `indigo_frame_preview_handle_ws()` are declared in `indigo_server_tcp.h` and implemented in `indigo_server_tcp.c`, alongside the HTTP/WebSocket server that owns the client sockets. There is no separate `indigo_frame_preview.h`.

At the point where the driver owns a completed, writable or read-only RGB24 frame, publish it only for a continuous stream:

```c
if (CCD_STREAMING_PROPERTY->state == INDIGO_BUSY_STATE) {
  indigo_frame_preview_publish_rgb24(
    rgb24_pixels,
    width * height * 3,
    width,
    height
  );
}
```

Requirements:

- `rgb24_pixels` starts at the first pixel, not at the start of an INDIGO buffer with a `FITS_HEADER_SIZE` prefix.
- Pixels are tightly packed, top-down, RGB order: `R0,G0,B0,R1,G1,B1,...`.
- The data must remain valid until `indigo_frame_preview_publish_rgb24()` returns. The function copies it into its WebSocket message before returning.
- Width and height must each fit in `uint16_t`.
- Do not publish Bayer, mono16, BGR, RGB48, JPEG, or a pointer with row padding as RGB24. Convert to tightly packed RGB24 first.
- Do not call star detection or `indigo_process_image()` merely to publish this stream.

`indigo_process_image()` may still run independently for normal INDIGO image/BLOB/file output. It is not part of the raw RGB24 WebSocket transport.

## EDS CCD Placement

For `ccd_eds`, publish from the normal full-frame branch of its camera `frame_handler`, after the RGB24 source buffer has been produced and before its ownership is released. Do not publish from `preview_frame_ready_callback()`: that callback is panel-only and publishes 160x160 gray16 plus centroid data.

Target flow:

```text
CCD_STREAMING command
  -> EDS continuous capture worker
  -> ccd_eds frame_handler receives full RGB24 frame
  -> indigo_frame_preview_publish_rgb24(...)
  -> /api/streaming/frame
  -> imager.html canvas
```

The EDS driver must not change Imager Agent properties to send the pixels. The agent already forwards `CCD_STREAMING` to the selected CCD and reports its state.

## Current Simulator Reference

`indigo_drivers/ccd_simulator/indigo_ccd_simulator.c` publishes the simulator DSLR's RGB24 frame when `CCD_STREAMING` is Busy. This is the reference integration:

```c
indigo_frame_preview_publish_rgb24(raw, size, DSLR_WIDTH, DSLR_HEIGHT);
```

## Browser Rendering

`indigo_server/resource/imager.html` opens `/api/streaming/frame` when its Preview button starts `CCD_STREAMING`. It validates the header and message length, copies RGB triplets to canvas RGBA pixels, and lets normal responsive CSS fit the canvas to the image container. It intentionally does not use nearest-neighbor panel scaling or centroid overlays.

## Concurrency and Client Lifetime

The publisher holds a fixed table of `FRAME_PREVIEW_MAX_CLIENTS` (currently 8) slots. A connection arriving when every slot is taken still completes the HTTP 101 upgrade and is then closed immediately, with no frame and no WebSocket close frame. A browser therefore cannot distinguish "server full" from an ordinary disconnect, and must treat an empty short-lived connection as a retryable condition rather than a fatal one.

The socket is server-to-client only, but that is a convention, not an enforced protocol. Anything a client sends is read and discarded without WebSocket frame parsing, so a client close frame is seen only as a stream end. There is no ping/pong keepalive: an idle NAT or proxy timeout will silently drop the connection, and the client is responsible for noticing the stall and reconnecting.

Client disconnection never affects acquisition. The camera keeps running and the driver keeps publishing whether or not anyone is attached; `indigo_frame_preview_publish_rgb24()` returns immediately when no client is connected, before building a frame at all.

There is no authentication or origin check on `/api/streaming/frame`. Any client that can reach the server port can attach and receive live camera pixels. Access control is expected to come from network-level isolation.

## Operational Limits

The frame is uncompressed. A 1600x1200 frame is 5,760,000 bytes before WebSocket framing.

`indigo_frame_preview_publish_rgb24()` never touches the network and never blocks on a client. Each client slot holds a one-frame mailbox. The publisher builds the WebSocket frame once, stores it into every attached client's mailbox — overwriting and freeing any frame that client has not yet picked up — signals the client's condition variable, and returns. The last recipient takes ownership of the buffer, so a single-client session performs no extra copy.

Each client's own thread drains its mailbox and does the blocking `send()`. A client that stops reading therefore falls behind alone: its mailbox keeps getting overwritten and it simply misses frames, while every other client and the driver thread proceed at full rate. Peak memory for a stalled client is one frame.

A blocked `send()` is bounded by the server-wide `SO_SNDTIMEO` of 5 seconds applied to every accepted connection. A timeout leaves the WebSocket stream truncated mid-frame with no way to resynchronise, so that client is dropped. Because the socket is owned exclusively by its own thread, the publisher is never affected. Between frames each thread also polls its socket with a non-blocking `recv()`, so a client that disconnects without ever sending anything is reaped promptly instead of lingering until a send finally fails.

Production camera integration should measure capture latency and network throughput at its intended ROI and exposure settings.
