# 10 — Audio Output

Getting sound out, and why it crackles.

---

## 1. Output basics

```c
int port = sceAudioOutOpenPort(SCE_AUDIO_OUT_PORT_TYPE_MAIN,
                               256,          /* grain, samples per call */
                               48000,        /* rate */
                               SCE_AUDIO_OUT_MODE_STEREO);
sceAudioOutSetVolume(port, SCE_AUDIO_VOLUME_FLAG_L_CH | SCE_AUDIO_VOLUME_FLAG_R_CH,
                     (int[]){ 32767, 32767 });

/* blocking submit of 256 stereo frames */
sceAudioOutOutput(port, buffer);
```

Key facts:

- granularity is the grain size you opened with; too small = overhead, too large =
  latency and risk of underrun
- rates commonly used: 48000, 44100, 32000. Melee's port mixes at **32000** to save
  bandwidth; PCSX AwO opens at 44100
- the output call blocks until the buffer is consumed — this is your pacing hook
- `SCE_AUDIO_OUT_MODE_MONO` and `..._STEREO` are the practical modes

## 2. Buffering strategy

```
 mix thread ──▶ ring buffer ──▶ sceAudioOutOutput ──▶ hardware
```

- preallocate every buffer; never `malloc` in the audio path
- keep 2–3 grains in flight
- if the mixer falls behind, output silence rather than stale data (Melee keeps a
  3-frame fallback)
- raise the audio thread priority above the render thread

## 3. Codecs available in hardware

| Path | Use |
|------|-----|
| Raw PCM via `SceAudioOut` | everything you mix yourself |
| `SceAtrac` (ATRAC9) | streamed music; the hardware decoder retail titles use |
| `SceAudiodec` | ATRAC9 / AAC / MP3 decode-to-PCM in software |
| `SceAudioIn` | microphone |

For a port, the usual split is: **music as ATRAC9 through `SceAtrac`**, everything
else mixed as PCM.

## 4. Software mixing at scale

Melee's port is the existence proof that this CPU can do it:

| Parameter | Value |
|-----------|-------|
| voices | 64 |
| rate | 32 kHz |
| frame | 160 samples |
| formats | ADPCM4, PCM16, PCM8 |
| extra | loop points, per-voice predictor state, ARAM block allocator |

It reproduces a hardware audio DSP in software and still holds frame rate. That is
the ceiling of ambition for audio emulation here — and it is enough.

## 5. Why it crackles

| Cause | Fix |
|-------|-----|
| audio thread lower priority than render | raise it |
| frame time spikes > grain duration | bound frame time, add a buffer |
| `malloc` in the mix path | preallocate |
| opening too small a grain | increase grain |
| decoding on the audio thread | decode on the stream thread |
| rate mismatch vs. hardware | resample once at load, not per frame |

## 6. Loudness and clipping

Mix in float or 32-bit int, clamp once at submission. Clamping per voice, or
letting sums exceed int16, produces distortion that sounds like a completely
different bug.

## 7. Sources

- `Other Ports/melee-Vita-vita-port/platforms/vita/game/audio.c` (AX/ARAM emulation)
- `Other Emulators/PCSX_AwO-master/frontend/vita/standalone.c` (SDL3 audio stream)
- `Sdk/include/psp2/audioout.h`, `audioin.h`, `atrac.h`, `audiodec.h`
