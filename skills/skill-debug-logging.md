---
name: debug-logging
description: Get a usable log off the console — file logs, UDP loggers, on-screen overlays, and the log plugins that work on apps you cannot rebuild.
when: Something fails and you have no visibility, or you need logs from hardware with no PC attached.
---

# Skill: Debug Logging

## Step 1 — Always have a file log

```c
auto LogWrite(LogLevel level, const char* fmt, ...) noexcept -> void {
    FILE* f = g_log_file();          /* ux0:data/<game>/log.txt, opened once */
    if (f == nullptr) return;
    va_list args; va_start(args, fmt);
    vfprintf(f, fmt, args);
    va_end(args);
    fputc('\n', f);
    fflush(f);                        /* per line, on purpose */
}
```

Flushing every line is wasteful and correct: a crash keeps the last line written.

## Step 2 — Add an on-screen overlay

Keep a ring buffer of the last ~20 lines and draw them with `vita2d` text in a
corner, toggled by a button chord. Works with no network, survives most crashes, and
is the fastest way to see what a boot sequence did.

Play! ships one (`Source/ui_vita/debug_screen.cpp` + `font.c`); PrincessLog has a
self-contained GXM debug screen with an embedded font blob.

## Step 3 — Add UDP when you need volume

```c
int sock = sceNetSocket("log", SCE_NET_AF_INET, SCE_NET_SOCK_DGRAM, 0);
/* send to LOG_HOST:18197 */
```

Watch on the PC with a UDP listener. Melee uses 18197. Requires `SceNet_stub` +
`sceNetInit`.

## Step 4 — If you cannot rebuild the app

Use a logging plugin:

| Tool | What it gives you |
|------|-------------------|
| `Tools/PrincessLog` | kernel module + PC receiver: network logging for any app |
| `Tools/PSMLogUSB` | ring buffer over USB |
| `Tools/ShipLog2.0` | alternative logger with PC back end |

These are how you observe a retail title or a third-party binary.

## Step 5 — Log the right things

| Category | Example |
|----------|---------|
| boot gates | "kubridge ok", "archive ok", "shader compiler ok" |
| resource loads | path, bytes, ms |
| renderer | draw count, shader cache tier hit |
| errors | every negative return value, with the call name |
| state changes | resolution switch, relaunch, save/load |

## Step 6 — Release builds strip it

Release = `-O3 -DNDEBUG`, no logging, no debugger, no symbols. Put diagnostics behind
explicit build switches so they cannot leak into a shipped build by accident.

## Diagnosis

| Symptom | Cause |
|---------|-------|
| log stops at the same line | crash in the next call — add logging around it |
| log file empty but app ran | writing to `app0:` (read-only) |
| UDP log never arrives | `sceNetInit` not called, or port blocked |
| log slows the game | flushing per line in a hot path; keep verbose logging to a debug flag |

## Never

- never write logs to `app0:`
- never log every frame in a release build
- never ship a build without saving its `.elf` — a log is useless without symbols
