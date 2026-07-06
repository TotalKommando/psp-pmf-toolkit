# psp-pmf-toolkit

**Was created/generated using Claude to figure out why @Blue's ARK-V's new animated ICON1.PMF files were glitching out or blacking out**

Convert any video into a **working `ICON1.PMF`** for PSP homebrew EBOOTs — the
animated icon that plays in the XMB Game menu. Byte-matched to Sony's own icon
format by reverse-engineering official samples, and **verified on real PSP
hardware**.

Also includes a PSMF/AT3 analyzer and a pure-Python EBOOT.PBP repacker so you
can embed the icon (and sound) without any SDK.

> `SND0.AT3` creation is included but **experimental / not yet working** — see
> [Audio status](#audio-status-snd0at3) below.

## Requirements

- Python 3.8+ (no packages needed — stdlib only)
- [ffmpeg](https://ffmpeg.org/) (`ffmpeg` + `ffprobe` on PATH), any recent build with libx264

## Quick start

```sh
# MP4 (or anything ffmpeg reads) -> XMB-ready animated icon
python mp42pmf.py convert input.mp4 -o ICON1.PMF

# inspect any PMF or AT3 and check it against the known XMB limits
python mp42pmf.py analyze ICON1.PMF
python mp42pmf.py analyze some_game_icon1.pmf

# embed into an existing homebrew EBOOT.PBP (no SDK needed)
python mp42pmf.py pbp MyApp/EBOOT.PBP -o EBOOT.PBP --icon1 ICON1.PMF --title "MY APP"
```

`convert` handles everything: scaling/padding to 144×80, frame-rate conversion
to 29.97 fps, the constrained two-pass H.264 encode, PSMF muxing, and a full
self-verification pass (structure diff, index-table validation, round-trip
decode). If the result would exceed the XMB size budget it automatically
re-encodes at a lower bitrate.

Options:

| flag | default | meaning |
|---|---|---|
| `--budget-kb` | 450 | target file-size budget (XMB limit is ~500 KB, shared with SND0.AT3 if present) |
| `-o` | `ICON1.PMF` | output path |

## Why the PMF conversion was glitching out

The XMB icon player is **not** a general PMF player. It has a small fixed
ring buffer sized for Sony's icon encoding regime. Converters that produce
structurally-valid PSMF at movie-grade settings (high bitrate, big VBV buffer)
play as jumbled/glitchy frames or black. A working icon must match the regime:

| parameter | required value |
|---|---|
| resolution | 144×80 |
| codec | H.264 **Main@2.1**, CABAC, 1 ref frame, no B-frames |
| frame rate | 29.97 (30000/1001) |
| HRD (VBV) | **600 kbps rate, 600 kbit buffer** |
| P-STD buffer declared | **80 KB** (appears in 3 places in the mux) |
| color | full range flag set, no colour description |
| file size | < ~500 KB (shared budget with SND0.AT3) |

The muxer in this tool reproduces Sony's exact container layout: 2048-byte
packs, per-GOP `0xBF` frame-index packets, sparse PTS stamping, padding
behavior, header TLV — the works. See [FINDINGS.md](FINDINGS.md) for the full
reverse-engineered format documentation.

## PBP repacker

```sh
python mp42pmf.py pbp <donor EBOOT.PBP> -o EBOOT.PBP \
    --icon1 ICON1.PMF [--snd0 SND0.AT3] [--title "NEW TITLE"]
```

Takes any existing homebrew EBOOT.PBP, swaps in the new media entries and
optionally patches the `PARAM.SFO` title, and writes a fresh PBP. Pure Python,
no SDK. The XMB plays ICON1/SND0 as soon as the entry is highlighted — the
executable is untouched.

## Audio status (SND0.AT3)

Not solved yet. Findings so far (all verified on hardware):

**AT3 was apparently already figured out by @GrayJack so I left it here**

- Every known-good `SND0.AT3` (from retail games and homebrew) is
  **ATRAC3+, 48–128 kbps, 44.1 kHz stereo** in a RIFF wrapper with `fact` +
  optional `smpl` loop chunks.
- Plain ATRAC3 (~66 kbps) does **not** play, despite what several guides claim.
- The only open-source encoder (atracdenc) can only produce ATRAC3+ at a
  hardcoded 352.8 kbps, which the PSP rejects.
- The RIFF wrapper code in this tool is proven correct (it reproduces a Sony
  sample byte-for-byte from raw frames).

**Bottom line:** producing a working SND0.AT3 currently requires Sony's
`at3tool.exe` (from the PSP SDK, common in PSP modding toolkits). Drop it next
to `mp42pmf.py` and run `python mp42pmf.py snd0 input.wav -o SND0.AT3` — the
tool auto-detects it and mirrors the settings real game SND0s use
(`-e -br 64 -wholeloop`).

## Subcommand reference

```
analyze  <file.pmf|file.at3>          dissect + validate against XMB limits
convert  <video> [-o out] [--budget-kb N]   video -> ICON1.PMF
snd0     <audio> [-o out] [--codec at3p|at3] [--encoder auto|at3tool|atracdenc]
all      <video> <audio> [--outdir D]  both at once + combined budget check
pbp      <donor.pbp> [-o out] [--icon1 F] [--snd0 F] [--title S]
```
