# 11 — Video & Media

Playing movies on a Vita, and why the answer is almost always "transcode first".

---

## 1. What the hardware decodes

| Path | Format | Notes |
|------|--------|-------|
| `SceAvPlayer` | H.264 in MP4, AAC audio | the practical path for cutscenes |
| `SceVideoDec` | H.264 elementary stream | lower level, more control, more work |
| `SceMp4` | MP4 container parsing | pair with `SceAvPlayer` or `SceVideoDec` |
| `SceJpeg` / `SceJpegArm` | JPEG | fast stills / frame decode |
| `SceMp4Rec` | encode | capture, not playback |

There is **no** Vita decoder for Bink, VP6, Theora, MPEG-2, or anything the 6th
console generation used. If the source game used those, transcode.

## 2. SceAvPlayer skeleton

```c
sceAvPlayerInit(&player, &params);                 /* mem, file, video, audio, event callbacks */
sceAvPlayerAddSource(player, "ux0:data/game/movies/intro.mp4");
sceAvPlayerSetLooping(player, false);
sceAvPlayerStart(player);
/* per frame: sceAvPlayerGetVideoData(player, &frame) → upload frame->pData to a texture */
```

Practical constraints:

- video frame dimensions come from the file; a 1920×1080 source will not play
  smoothly. Target **960×544** for full-screen video, less for in-world screens.
- audio through the player's own path, or decode separately if you need it mixed
  with game audio
- the player reads through FIOS — put movies on `ux0:`, not inside a huge VPK

## 3. Transcoding pipeline (PC side)

```
 Bink / THP / other  ──ffmpeg──▶  H.264 High, 960x544, 1.5–2.5 Mbps
                                  AAC-LC 96–128 kbps stereo
                                  +faststart, yuv420p
```

`ffmpeg -i in.bik -vf scale=960:544 -c:v libx264 -profile:v high -b:v 2M \
 -c:a aac -b:a 128k -movflags +faststart out.mp4`

Sanity requirements for the Vita decoder: baseline/main/high profile, yuv420p,
no interlacing, and a level the hardware supports. If playback fails instantly, the
profile/level is the first thing to change.

## 4. Size math matters more than quality

Example from the Arkham target: 78 Bink files totalling **2.9 GB** became roughly
**0.6 GB** after transcoding — the single largest size win available in the whole
project. Video is where you buy back your storage budget, and where "keep the
original quality" has to lose.

Offer it as a download choice: a base pack with low-bitrate movies and an optional
"HD movies" pack.

## 5. Still images and screenshots

- `SceJpeg` decodes JPEG in hardware — fast texture loading for a port that
  ships photographic assets
- `SceScreenshot` takes the system screenshot; useful for bug reports
- for video thumbnails, decode one frame and cache it

## 6. Common failures

| Symptom | Cause |
|---------|-------|
| player starts, no video | texture upload path wrong; frame data is YUV, not RGBA |
| audio but black screen | frame timestamps ignored; you must present on the player's clock |
| stutter every few seconds | the file is being read from a slow path or is too high bitrate |
| instant exit | unsupported profile/level |
| drift between video and audio | presenting frames on your own frame counter instead of the player's |

## 7. Sources

- `Sdk/include/psp2/avplayer.h`, `videodec.h`, `mp4.h`, `jpeg.h`
- `Other Ports/raider-vita-master/loader/fmv.c` + its `ffmpeg` transcode scripts
- `Other Ports/melee-Vita-vita-port/platforms/vita/game/thp.c` (software THP decode)
