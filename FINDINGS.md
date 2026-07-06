# PSP ICON1.PMF / SND0.AT3 — reverse-engineering findings

Everything below was derived by byte-level analysis of official Sony samples
(a retail game's `icon1.pmf` and several games' `SND0.AT3`), cross-checked
against a broken third-party conversion, and validated on real PSP hardware.

## 1. ICON1.PMF (PSMF video)

### 1.1 Why converters fail

A structurally valid PSMF is not enough. The XMB icon player has a small fixed
ring buffer sized for Sony's icon regime. A typical failed conversion (movie
settings) vs. the Sony reference:

| | Sony icon (works) | typical broken conversion |
|---|---|---|
| HRD in SPS VUI | 600 kbps / 600 kbit CPB | 2 Mbps / 2 Mbit CPB |
| P-STD buffer declared | 80 KB | 251 KB |
| file size | well under 500 KB | often far over |
| symptom | — | jumbled/glitchy frames, or black |

### 1.2 Elementary stream (H.264)

- 144×80, yuv420p, 29.97 fps (timing 4004/240000 in VUI, fixed_frame_rate=1)
- Profile 77 (Main), level 21, CABAC, `ref_frames=1`, no B-frames
  (`pic_order_cnt_type=2`), `frame_mbs_only=1`, `weighted_pred=0`
- VUI: `video_full_range_flag=1`, **no** colour description, `pic_struct=1`,
  both `nal_hrd` and `vcl_hrd` present with bitrate=cpb=600000
- Every access unit: AUD (`09 10` for I, `09 30` for P) + SEI (buffering
  period at IDR, pic timing everywhere) + slice. SPS+PPS repeated at each IDR.
- GOPs: IDR-led, scene-adaptive, ~27–57 frames in the Sony sample.

x264 reproduces all of this with: `profile=main level=2.1 ref=1 bframes=0
weightp=0 keyint=48 min-keyint=8 scenecut=40 vbv-maxrate=600 vbv-bufsize=600
nal-hrd=vbr aud=1 range=pc` (2-pass; strip SEI type 5 / filler NALs from the
output).

### 1.3 PSMF container

File = `0x800`-byte header + stream (multiple of 2048).

Header (big-endian; all timestamps 90 kHz):

| offset | content |
|---|---|
| 0x00 | `"PSMF0014"` |
| 0x08 | u32 header size = 0x800 |
| 0x0C | u32 stream size (file size − 0x800) |
| 0x50 | u32 0x3E (size of the block that follows) |
| 0x54 | 6-byte firstTS = **90000** |
| 0x5A | 6-byte lastTS = **90000 + 3003 × numFrames** |
| 0x60 | u32 25000 (mux rate, 50-byte units = 10 Mbps) |
| 0x64 | u32 90000 (timescale) |
| 0x68 | `01 01` |
| 0x6A | u32 0x24, then firstTS/lastTS again (6+6), u16 1, u32 0x12 |
| 0x80 | u16 numStreams = 1 |
| 0x82 | stream entry: `E0 00 20 50 00×8 W/16 H/16 00 00` (`20 50` mirrors the 80 KB P-STD) |
| 0x92+ | zeros — **no EP map** in icon PMFs |

Stream: strict 2048-byte packs, each = 14-byte MPEG-2 pack header
(`mux_rate=25000`, stuffing=0) + one PES packet (+ trailing padding PES).

- Pack 0 additionally carries the 18-byte system header, verbatim:
  `00 00 01 BB 00 0C 80 C3 51 80 F0 7F B9 E0 50 BD E0 08`
  (`E0 50` = 80 KB video P-STD bound — second appearance).
- **Chunks = IDR GOPs.** Each chunk starts on a pack boundary with a
  private-stream-2 PES (`0xBF`) holding a frame-size index:
  `01 E0` + six u16 header fields + u16 tableBytes(=2+4n) + u16 n +
  n × (u16 flag, u16 auSize). Flag is `0x0080` except `0x0000` on the last
  entry. Header fields: `(0,0,1,1,0,0)` first chunk, `(0,1,1,2,0,0)` after
  (semantics unknown; hardware tolerates variation).
- Video PES: stream id `0xE0`, byte7=`0x81`. PTS+DTS (DTS = PTS − 3003)
  stamped on (a) the first PES of every chunk — flags `0xC1`, 13-byte header
  including PES extension `1E 60 50` (P-STD 80 KB, third appearance) — and
  (b) any PES whose first contained AU start is ≥16 frames past the last
  stamp — flags `0xC0`, 10-byte header. All other PES: flags 0, no header
  fields. The stamped PTS always belongs to the **first AU that starts inside
  that PES payload**.
- Chunk ends: padding PES `0xBE` filled with `0xFF` closes the last pack of
  each chunk (1–5 leftover bytes become PES header stuffing instead).
- SCR advances monotonically (Sony uses a VBV-style just-in-time schedule;
  a linear ramp from `DTS₀ − 76850` works — hardware doesn't appear to check).
- No MPEG program-end code.

PTS grid: frame *i* has PTS `90000 + 3003·i`. Max gap between stamped PTS
must stay < 0.7 s (Sony's worst is 0.6 s).

### 1.4 Budgets

- Keep the file < ~500 KB; if a `SND0.AT3` is present the *combined* size
  shares the budget (folklore says 500 KB combined; a 471 KB retail SND0
  exists alone, so treat as soft — but stay under it when possible).

### 1.5 Verification

`mp42pmf.py analyze` re-derives all of the above from a file: header fields,
pack alignment, PES pattern, BF table vs. actual AU sizes, SPS/PPS/HRD, PTS
gaps, size budget. `convert` also does a demux round-trip (must be
byte-identical to the encoder output) and a full ffmpeg decode (frame count
must match). This pipeline is confirmed working on real hardware.

## 2. SND0.AT3 (status: unsolved encoding gap)

### 2.1 Known-good samples (4 analyzed, all ATRAC3+)

| source | rate | frame size | delay | loop chunk |
|---|---|---|---|---|
| retail game icon sound (Sony sample) | 64.768 kbps | 376 B | 2820 | yes |
| GTA Vice City Stories | 48.232 kbps | 280 B | 3311 | yes |
| OutRun 2006 (471 KB, 80 s!) | 48.232 kbps | 280 B | 2048 | **no** |
| MotorStorm Arctic Edge / BatterySteve homebrew | 128.168 kbps | 744 B | 2704/3216 | yes |

### 2.2 RIFF wrapper (fully understood)

`RIFF/WAVE` with:
- `fmt ` (52 B): WAVE_FORMAT_EXTENSIBLE, 2ch 44100, avg bytes/sec, blockAlign
  = frame bytes, samplesPerBlock 2048, ATRAC3+ GUID
  `E923AABF-CB58-4471-A119-FFFA01E4CE62`, ext `01 00 28 XX 00…` where
  **XX = blockAlign/8 − 1** (validated across 280/376/744-byte frames; the
  `0x28` byte is constant)
- `fact` (8 B): total PCM samples, encoder delay
- `smpl` (60 B, optional): standard sampler chunk, one loop,
  start = delay, end = delay + samples − 1
- `data`: raw ATRAC3+ frames, no sync headers

The wrapper implementation reproduces the Sony sample **byte-for-byte** from
its raw frames.

### 2.3 The fact-capacity invariant (probable make-or-break rule)

Every known-good file obeys:

```
fact.samples + fact.delay + 368 <= n_frames * samples_per_frame
```

Four of five at3tool files have a tail margin of **exactly 368 samples**
(the non-looping one has a plain ceil margin). Every file that failed on
hardware **overclaimed** — its `fact` promised more samples than the frames
can decode (even by as little as 309). Even known-good GTA *frames*, trimmed
and rewrapped with an overclaiming `fact`, go silent. The tool now clamps
`fact.samples` to `capacity - delay - 368` (`clamp_fact_samples`).

### 2.4 Hardware test results

| attempt | fact fits? | result |
|---|---|---|
| ATRAC3+ 352.8 kbps (atracdenc) — 574 KB and 456 KB combined | no (−309) | silent |
| plain ATRAC3 66 kbps (atracdenc LP4) | no (−561) | silent |
| GTA frames trimmed to 5 s, naive fact | no (−3311) | silent |
| real GTA SND0.AT3 unchanged | yes | **plays** |
| v2 set (above three with clamped fact) | yes | pending |

If the v2 files still fail, fall back to the bitrate theory: only ATRAC3+
48–128 kbps plays, which requires Sony `at3tool.exe`
(`at3tool -e -br 64 -wholeloop in.wav SND0.AT3`). ffmpeg has no ATRAC3/
ATRAC3+ encoder; atracdenc's ATRAC3+ frame size is hardcoded to 2048 bytes.

## 3. EBOOT.PBP notes

- PBP = `\0PBP` + version + 8 u32 offsets: PARAM.SFO, ICON0.PNG, ICON1.PMF,
  PIC0.PNG, PIC1.PNG, SND0.AT3, DATA.PSP, DATA.PSAR. Entry size = next offset
  − own offset. Repacking is trivial (see `pbp` subcommand).
- XMB plays ICON1.PMF / SND0.AT3 when the entry is highlighted; the
  executable payload is irrelevant to icon/sound testing.
- Savedata folders also play SND0.AT3, so they're a convenient test bed.
