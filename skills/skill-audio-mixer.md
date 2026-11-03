---
name: audio-mixer
description: Get audio out of the Vita reliably — port setup, buffering, priorities, hardware codecs, and why crackle happens.
when: No sound, wrong sound, or crackle/dropouts under load.
---

# Skill: Audio Mixer

## Step 1 — Open a port

```c
int port = sceAudioOutOpenPort(SCE_AUDIO_OUT_PORT_TYPE_MAIN,
                               256,     /* grain: samples per submit */
                               48000,   /* rate */
                               SCE_AUDIO_OUT_MODE_STEREO);
sceAudioOutSetVolume(port, SCE_AUDIO_VOLUME_FLAG_L_CH | SCE_AUDIO_VOLUME_FLAG_R_CH,
                     (int[]){ 32767, 32767 });
```

Grain too small = call overhead; too large = latency and underrun risk. 256–512
samples is the usual range.

## Step 2 — Submit blocks and use them as pacing

```c
sceAudioOutOutput(port, buffer);   /* blocks until consumed */
```

That call is a natural frame limiter for audio-driven loops. Do not also sleep.

## Step 3 — Threading

- dedicated audio thread, priority **above** the render thread
- preallocate every buffer; no `malloc` in the mix path
- 2–3 grains in flight
- on starvation, output silence rather than stale data

## Step 4 — Choose the codec path

| Content | Path |
|---------|------|
| sound effects | decode to PCM at load, mix yourself |
| music / long streams | `SceAtrac` ATRAC9 (hardware decode) |
| voice | PCM or ATRAC, whichever fits |
| the original game's format | decode offline, re-encode once |

`SceAudiodec` gives software ATRAC9/AAC/MP3 decode if you need it in your own
pipeline.

## Step 5 — Mixing rules

- mix in float or int32, clamp once at submission
- per-voice gain then master gain; never clamp per voice
- resample offline, not per frame
- silence voices cleanly (fade 2–3 ms) or you get clicks

## Step 6 — Volume and persistence

Store master/music/sfx separately, persist to `ux0:data/<game>/settings.bin`, apply
in the mixer. Users will change these; losing them is a bug report.

## Step 7 — Diagnosis

| Symptom | Fix |
|---------|-----|
| crackle under load | raise audio thread priority, bound frame time, add a buffer |
| crackle only in menus | menu code doing IO on the audio thread |
| silence | port not opened, or volume left at 0, or buffer not submitted |
| wrong pitch | sample rate mismatch — resample at load |
| desync with video | present video on the player clock, not your frame counter |
| clicks on start/stop | no envelope; add a short fade |

## Step 8 — The scale test

If you need many simultaneous voices (game audio, not just one stream), Melee's
port is the reference: 64 voices, ADPCM/PCM16/PCM8, loop points, 32 kHz, in software,
while holding frame rate. If that budget is not enough, reduce voices — do not
reduce the audio thread's priority.

## Never

- never `malloc`, `fopen`, or log-heavily inside the audio callback
- never let the render thread's worst frame exceed your audio buffer depth
