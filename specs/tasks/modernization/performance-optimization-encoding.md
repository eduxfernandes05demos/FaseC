# Task: Performance Optimization — Frame Encoding

> Optimize frame encoding for streaming: pixel buffer extraction, video encoding pipeline.

---

## Metadata

| Field | Value |
|-------|-------|
| **Priority** | P2 — Cloud-Native |
| **Estimated Complexity** | High |
| **Estimated Effort** | 2 weeks |
| **Dependencies** | Headless video, Streaming gateway |
| **Blocks** | Production-quality streaming |
| **Phase** | Phase 4 — Cloud-Native |

---

## Objective

Build an optimized frame encoding pipeline that extracts rendered frames from the Quake engine's framebuffer and encodes them to H.264 or VP8 video with minimal latency (target: ≤ 16ms per frame at 60 FPS, ≤ 10ms at 30 FPS).

---

## Pipeline Architecture

```
                    ≤1ms         ≤1ms          ≤5-10ms         ≤1ms
┌────────────┐  ┌─────────┐  ┌──────────┐  ┌────────────┐  ┌──────────┐
│ Engine     │  │ Frame   │  │ Color    │  │  Video     │  │ Network  │
│ Framebuffer│─▶│ Extract │─▶│ Convert  │─▶│  Encode    │─▶│ Transmit │
│ (vid.buffer)│  │ (memcpy)│  │(RGB→YUV) │  │ (H.264)   │  │ (WebSocket)│
└────────────┘  └─────────┘  └──────────┘  └────────────┘  └──────────┘
```

---

## Stage 1: Frame Buffer Extraction

### Software Renderer

The software renderer writes directly to `vid.buffer` (palette-indexed 8-bit pixels):

| Property | Value | Source |
|----------|-------|--------|
| Buffer pointer | `vid.buffer` | `legacy-src/desktop-engine/vid.h` — `viddef_t` struct |
| Width | `vid.width` | Typically 320-640 |
| Height | `vid.height` | Typically 200-480 |
| Pixel format | 8-bit palette indexed | `vid.colormap` provides palette |
| Row stride | `vid.rowbytes` | May differ from width |
| Frame completion | After `SCR_UpdateScreen()` | `legacy-src/desktop-engine/screen.c` |

**Extraction method**:
```c
void extract_frame_software(uint8_t *rgb_out, int *out_width, int *out_height) {
    *out_width = vid.width;
    *out_height = vid.height;

    /* Convert palette-indexed → RGB24 */
    for (int y = 0; y < vid.height; y++) {
        uint8_t *src_row = vid.buffer + y * vid.rowbytes;
        uint8_t *dst_row = rgb_out + y * vid.width * 3;
        for (int x = 0; x < vid.width; x++) {
            uint8_t idx = src_row[x];
            dst_row[x*3 + 0] = host_basepal[idx*3 + 0]; /* R */
            dst_row[x*3 + 1] = host_basepal[idx*3 + 1]; /* G */
            dst_row[x*3 + 2] = host_basepal[idx*3 + 2]; /* B */
        }
    }
}
```

### OpenGL Renderer

The OpenGL renderer uses GPU framebuffer:

| Property | Value | Source |
|----------|-------|--------|
| Read method | `glReadPixels()` | `legacy-src/desktop-engine/gl_screen.c:618` |
| Pixel format | RGB (24-bit) | Direct RGB output |
| Performance | Slow (GPU→CPU stall) | Needs async readback |

**Extraction method** (slow path):
```c
void extract_frame_opengl(uint8_t *rgb_out, int *out_width, int *out_height) {
    *out_width = glwidth;
    *out_height = glheight;
    glReadPixels(0, 0, glwidth, glheight, GL_RGB, GL_UNSIGNED_BYTE, rgb_out);
    /* Note: OpenGL origin is bottom-left, need to flip vertically */
}
```

---

## Stage 2: Color Space Conversion

Convert RGB24 to YUV420p for video encoding:

```c
/* RGB24 → YUV420p conversion (SIMD-optimized) */
void rgb_to_yuv420(const uint8_t *rgb, int width, int height,
                    uint8_t *y_plane, uint8_t *u_plane, uint8_t *v_plane) {
    for (int j = 0; j < height; j++) {
        for (int i = 0; i < width; i++) {
            int r = rgb[(j*width + i)*3 + 0];
            int g = rgb[(j*width + i)*3 + 1];
            int b = rgb[(j*width + i)*3 + 2];

            y_plane[j*width + i] = ((66*r + 129*g + 25*b + 128) >> 8) + 16;

            if (j % 2 == 0 && i % 2 == 0) {
                int idx = (j/2)*(width/2) + (i/2);
                u_plane[idx] = ((-38*r - 74*g + 112*b + 128) >> 8) + 128;
                v_plane[idx] = ((112*r - 94*g - 18*b + 128) >> 8) + 128;
            }
        }
    }
}
```

**SIMD optimization**: Use SSE2/NEON intrinsics for bulk pixel conversion — expected 4-8x speedup.

---

## Stage 3: Video Encoding

### FFmpeg/libx264 Integration

```c
#include <libavcodec/avcodec.h>
#include <libavutil/frame.h>

typedef struct {
    AVCodecContext *codec_ctx;
    AVFrame *frame;
    AVPacket *pkt;
    int width, height;
    int frame_count;
} video_encoder_t;

int encoder_init(video_encoder_t *enc, int width, int height, int fps) {
    const AVCodec *codec = avcodec_find_encoder(AV_CODEC_ID_H264);
    enc->codec_ctx = avcodec_alloc_context3(codec);
    enc->codec_ctx->width = width;
    enc->codec_ctx->height = height;
    enc->codec_ctx->time_base = (AVRational){1, fps};
    enc->codec_ctx->framerate = (AVRational){fps, 1};
    enc->codec_ctx->pix_fmt = AV_PIX_FMT_YUV420P;
    enc->codec_ctx->gop_size = fps;  /* Keyframe every second */

    /* Low-latency settings */
    av_opt_set(enc->codec_ctx->priv_data, "preset", "ultrafast", 0);
    av_opt_set(enc->codec_ctx->priv_data, "tune", "zerolatency", 0);

    avcodec_open2(enc->codec_ctx, codec, NULL);
    enc->frame = av_frame_alloc();
    enc->pkt = av_packet_alloc();
    return 0;
}

int encoder_encode_frame(video_encoder_t *enc, uint8_t *yuv_data,
                          uint8_t **out, int *out_size) {
    /* Fill AVFrame with YUV data */
    enc->frame->data[0] = yuv_data;                                /* Y */
    enc->frame->data[1] = yuv_data + enc->width * enc->height;    /* U */
    enc->frame->data[2] = yuv_data + enc->width * enc->height * 5/4; /* V */
    enc->frame->pts = enc->frame_count++;

    avcodec_send_frame(enc->codec_ctx, enc->frame);
    int ret = avcodec_receive_packet(enc->codec_ctx, enc->pkt);
    if (ret == 0) {
        *out = enc->pkt->data;
        *out_size = enc->pkt->size;
        return 1;
    }
    return 0;
}
```

---

## Stage 4: Optimizations

### Double Buffering

Render to one buffer while encoding the other:

```c
static uint8_t *frame_buffers[2];
static int current_buffer = 0;

/* After engine renders: */
void on_frame_complete(void) {
    int encode_buffer = current_buffer;
    current_buffer = 1 - current_buffer;  /* Swap */

    /* Encoder thread encodes from encode_buffer */
    /* Engine renders next frame to current_buffer */
}
```

### Palette LUT

Pre-compute palette → RGB lookup table to eliminate per-pixel palette lookup:

```c
static uint32_t palette_rgb_lut[256]; /* Pre-computed R,G,B for each palette index */

void build_palette_lut(const uint8_t *palette) {
    for (int i = 0; i < 256; i++) {
        palette_rgb_lut[i] = (palette[i*3] << 16) | (palette[i*3+1] << 8) | palette[i*3+2];
    }
}
```

### Skip Unchanged Frames

Compare frame checksums to skip encoding identical frames:

```c
static uint32_t last_frame_crc = 0;

qboolean frame_changed(const uint8_t *buffer, int size) {
    uint32_t crc = CRC_Block(buffer, size); /* Uses existing CRC from crc.c */
    if (crc == last_frame_crc) return false;
    last_frame_crc = crc;
    return true;
}
```

---

## Files Affected

| File | Change Type | Description |
|------|------------|-------------|
| `legacy-src/desktop-engine/encode.c` | **New** | Frame encoding pipeline |
| `legacy-src/desktop-engine/encode.h` | **New** | Encoding API |
| `legacy-src/desktop-engine/color_convert.c` | **New** | RGB→YUV conversion with SIMD |
| `legacy-src/desktop-engine/vid_headless.c` | **Edit** | Add double-buffered frame extraction |
| `legacy-src/desktop-engine/screen.c` | **Edit** | Trigger encoding after `SCR_UpdateScreen()` |
| `legacy-src/desktop-engine/CMakeLists.txt` | **Edit** | Add FFmpeg/libx264 dependency |

---

## Performance Targets

| Resolution | FPS | Encode Time | Preset | Method |
|-----------|-----|-------------|--------|--------|
| 320×240 | 60 | ≤ 5ms | ultrafast | Software H.264 |
| 640×480 | 30 | ≤ 10ms | ultrafast | Software H.264 |
| 640×480 | 60 | ≤ 8ms | — | Hardware VAAPI |
| 1280×720 | 30 | ≤ 16ms | ultrafast | Software H.264 |
| 1280×720 | 60 | ≤ 10ms | — | Hardware NVENC |

---

## Testing Requirements

| Test | Description |
|------|-------------|
| `test_extract_software` | Software renderer frame extraction produces valid RGB |
| `test_color_convert` | RGB→YUV420 conversion is correct (golden test) |
| `test_encode_frame` | Single frame encodes to valid H.264 NAL unit |
| `test_encode_stream` | 100 frames encode to playable H.264 stream |
| `test_encode_latency` | Encoding time ≤ 16ms at 640×480 |
| `test_double_buffer` | No tearing or corruption with double buffering |
| `test_skip_unchanged` | Unchanged frames are detected and skipped |
| `bench_encode_throughput` | Measure max FPS at each resolution |

---

## Acceptance Criteria

- [ ] Frame extraction from software renderer produces valid pixels
- [ ] Color conversion RGB→YUV420 is correct (verified against reference)
- [ ] H.264 encoding produces playable video
- [ ] Encoding latency ≤ 10ms at 640×480 (ultrafast preset)
- [ ] Double buffering eliminates frame extraction stalls
- [ ] Palette LUT reduces per-pixel overhead
- [ ] Frame skip detection works for static scenes
- [ ] SIMD color conversion provides ≥ 4x speedup over scalar
- [ ] Encoded stream is decodable by browser MediaSource API
