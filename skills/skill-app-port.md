---
name: app-port
description: Port a non-game app or tool to the Vita — foreground-only constraints, path redirection, IME, dialogs, networking, codecs, and file-descriptor limits.
when: The target is a utility, media player, file manager, reader, or CLI tool rather than a game.
---

# Skill: App Port

## Step 1 — Pick the shape

| Target | Route |
|--------|-------|
| existing C/C++ tool, no graphics | native VitaSDK build, console output or simple 2D |
| Android app with native libs | `.so` loader (skill `android-so-port`) |
| Java app | TeaVM C backend (doc 38) — heavy, reconsider |
| media / streaming app | **prefer native** over the loader route |

For media specifically: mpv renders through GXM, FFmpeg uses the SCE hardware
decoders, curl needs a large physically contiguous pool plus `CURLOPT_NOSIGNAL`.
That stack is proven natively and is painful through a loader.

## Step 2 — Accept foreground-only

There is no background execution. Home button suspends the process; sockets die and
files go stale.

- chunk long jobs and checkpoint them to `ux0:data/<app>/`
- reconnect on resume, never assume a live socket
- put "no background mode" in the user README

## Step 3 — Paths

| Kind | Prefix |
|------|--------|
| bundle (read-only) | `app0:` |
| state | `ux0:data/<app>/` |
| more room | `uma0:` / `imc0:` |

Redirect for Android-derived apps: `/sdcard/` → `ux0:data/<app>/`,
`/proc/cpuinfo` and `/proc/meminfo` → bundled text, `/dev/urandom` → Vita PRNG or a
seeded pool file.

## Step 4 — Input and text

- front touch reports **1920×1088**; scale to 960×544
- rear touch: Vita 1000 only, none on PS TV
- no L2/R2/L3/R3 on a handheld — map or drop them
- deadzone ~±16 counts on sticks
- text fields use `SceIme`; never raw scan codes

## Step 5 — Dialogs and permissions

- `SceCommonDialog` for pickers and message boxes
- Android runtime permissions have no equivalent — read from your data dir, no prompt
- if the app checks Play Services, stub it **and log it**, never fake success silently

## Step 6 — Networking

```
sceSysmoduleLoadModule(SCE_SYSMODULE_NET) → sceNetInit → sockets
```

HTTP(S) via `SceHttp`/`SceSsl`. Test on hardware; the emulator's network proves
nothing about the device.

## Step 7 — Respect the limits

| Limit | Value |
|-------|-------|
| simultaneous open files | ~64 |
| card writes | wear — batch small writes |
| memory | ~365 MiB with `ATTRIBUTE2=12` |

Serve large asset sets from RAM or one archive; close files aggressively.

## Step 8 — Audio for media apps

Output: `sceAudioOutOpenPort` + `sceAudioOutOutput` from a dedicated decode thread,
small ring of per-channel buffers, volume clamped to `SCE_AUDIO_OUT_MAX_VOL`.

Static-link the light codec set rather than pulling FFmpeg: `libopus`,
`libvorbis`/`libvorbisfile`, `libogg`, `libxmp-lite`, `dr_flac.h`, `mpg123`, `libmac`.

There is no `AudioTrack`, `OpenSL ES` or `AAudio` — a loader port must stub them.

## Step 9 — UI: draw only if you must

Text-and-lists UIs are cheaper as a software raster blitted through `sceDisplay`
than as a GLES app. Use vitaGL only when the UI genuinely needs GPU work.

## Step 10 — Verify

1. boots with no data directory present (creates it)
2. survives suspend/resume
3. closes cleanly and reopens its own files
4. works with a nearly-full card
5. `check_vpk.py` passes

## Never

- never write into `app0:`
- never assume the emulator's file descriptor limit matches hardware
- never port an Android `EditText` — use `SceIme`
