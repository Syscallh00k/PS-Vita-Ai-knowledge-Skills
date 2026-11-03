# 32 — Porting Apps and Tools (not games)

The Vita is also a fine target for utilities: media players, file managers, readers,
CLI tools. The constraint set is different from games.

> Derived from [yannis100/vita-port-skill](https://github.com/yannis100/vita-port-skill)
> (`skills/vita-app-port/references/app-runtime.md`, MIT). See [CREDITS](../CREDITS.md).

---

## 1. Foreground only

There are no background services. Home button / LiveArea suspension pauses the
process; when it resumes, sockets are dead and files may be stale.

Consequences:

- long jobs (downloads, index building, thumbnail caches) must be **chunked and
  resumable**, checkpointed into `ux0:data/<app>/`
- say this in the user README — "runs in background" is not a thing
- reconnect rather than assume a socket survived

## 2. Paths and Android redirects

| Kind | Prefix |
|------|--------|
| read-only bundle | `app0:` |
| writable state | `ux0:data/<app>/` |
| higher capacity | `uma0:` / `imc0:` |

For Android apps, redirect these in the IO wrapper:

| Android | Vita |
|---------|------|
| `/proc/cpuinfo`, `/proc/meminfo` | bundled text file (or generate at first run) |
| `/dev/urandom` | Vita PRNG or a seeded pool file |
| `/sdcard/`, `/storage/emulated/0/` | `ux0:data/<app>/` |
| asset paths | `ux0:data/<app>/assets/` via the AAssetManager bridge |

A file that reports plausible contents is often enough; apps read `/proc/cpuinfo`
for a core count they then clamp anyway.

## 3. Input

- front touch: the panel reports in **1920×1088**; scale to 960×544
- rear touch exists on the Vita 1000 only; PS TV has none
- no L2/R2/L3/R3 on a handheld; map or drop them in settings
- stick centre deadzone ~±16 counts before scaling
- keep confirm on Cross, back on Circle — do not inherit another platform's layout

## 4. Text entry

Use `SceIme` for any text field. It drives the on-screen keyboard and works with a
hardware keyboard on PS TV. Do not fall back to raw scan codes for text input, and
do not port a full Android `EditText` for multi-line cases — a Vita-native text
control is less work and behaves correctly.

## 5. Dialogs and permissions

- file pickers / message boxes: `SceCommonDialog` is the sanctioned path
- Android runtime permissions have **no** Vita equivalent — read from
  `ux0:data/<app>/` without prompting, and document that prompts are not part of the
  platform model
- if the app insists on a Play Services check, stub it **and log it**; do not
  silently fake a success the app depends on

## 6. Networking

- load `SCE_SYSMODULE_NET`, init `SceNet` before any socket call
- HTTP(S): `SceHttp` / `SceSsl`; the Vita has its own certificate store
- POSIX wrappers over `SceNet` cover most socket code
- test on hardware — Vita3K's network does not prove device behaviour

## 7. Storage and file-descriptor limits

| Limit | Value | Consequence |
|-------|-------|-------------|
| simultaneous open files | low (~64) | serve large asset sets from RAM or close aggressively |
| card writes | wear | batch and cache small writes |
| memory | ~365 MiB with `ATTRIBUTE2=12` | budget heap and VRAM explicitly |

For a CLI tool with no graphics, a 64–128 MiB heap is usually plenty.

## 8. Media apps: the proven stack

Output: `sceAudioOutOutput` from a dedicated decode thread with a small ring of
per-channel buffers; volume via `sceAudioOutSetVolume`
(`SCE_AUDIO_VOLUME_FLAG_L_CH | R_CH`, clamp to `SCE_AUDIO_OUT_MAX_VOL`).

Codecs — static-link the light set instead of pulling FFmpeg:

```
libopus · libvorbis/vorbisfile · libogg · libxmp-lite
dr_flac.h (header only) · mpg123 · libmac
```

There is **no Android audio HAL** on Vita: no `AudioTrack`, no `OpenSL ES`, no
`AAudio`. A `.so` loader port must stub or bridge them; a native recompile avoids
the problem entirely.

For a streaming client, prefer native over the loader route: mpv rendering through
GXM (`MPV_RENDER_API_TYPE_GXM`), FFmpeg using the SCE hardware decoders
(`aac_vita` / `mp3_vita` / `h264_vita`), and curl given a large physically
contiguous pool (`sceClibMspace`, ~20 MiB) plus `CURLOPT_NOSIGNAL`.

## 9. Reference apps worth reading

| Class | Example | Why |
|-------|---------|-----|
| native music player | joel16/ElevenMPV, GrapheneCt/ElevenMPV-A | `sceAudioOut` ring buffer + bundled codec libs |
| `libvita2d` UI fork | Perroabuelo/Material-Eleven | screen-off BGM playback, IME search |
| streaming client | dragonflylee/switchfin | borealis + NanoVG + mpv-on-GXM + hw decoders |
| `.so` loader tool | PKForge-style managers | AAssetManager + touch only — simplest loader app |

Note: "ElevenMPV" is a native Vita music player, not the `mpv` media player.
Name collisions in this ecosystem are real.

## 10. Draw, or do not draw

If the UI is text and lists, a software raster into a framebuffer plus a
`sceDisplay` blit can be simpler and cheaper than standing up GLES. Reach for
vitaGL only when you actually need GPU work.
