# 14 — Debugging & Logging

You will spend more time here than in the renderer. Set it up first.

---

## 1. The four logging channels

| Channel | Setup cost | Survives crash | Use |
|---------|-----------|----------------|-----|
| File log (`ux0:data/<game>/log.txt`) | none | mostly | always-on in dev builds |
| UDP log to PC | low | no | live tail, high volume |
| On-screen text overlay | low | yes | no PC attached |
| GDB remote | high | n/a | stepping, breakpoints |

Do file + on-screen from day one. Every reference project ends up with both.

## 2. Minimal logger

```c
typedef enum { LOG_INFO, LOG_WARN, LOG_ERROR } LogLevel;

[[nodiscard]] auto g_log_file() noexcept -> FILE*;

auto LogWrite(LogLevel level, const char* fmt, ...) noexcept -> void {
    FILE* f = g_log_file();
    if (f == nullptr) return;
    va_list args;
    va_start(args, fmt);
    vfprintf(f, fmt, args);
    va_end(args);
    fputc('\n', f);
    fflush(f);              /* flush per line: a crash keeps the last line */
}
```

`fflush` on every line is wasteful and correct. A log that stops one call before the
crash is the entire point.

## 3. UDP logging

Send datagrams to the PC and watch with a listener:

```c
int sock = sceNetSocket("log", SCE_NET_AF_INET, SCE_NET_SOCK_DGRAM, 0);
SceNetSockaddrIn addr = { .sin_family = SCE_NET_AF_INET,
                          .sin_port = sceNetHtons(18197),
                          .sin_addr.s_addr = sceNetHtonl(LOG_HOST) };
sceNetSendto(sock, line, strlen(line), 0, (SceNetSockaddr*)&addr, sizeof(addr));
```

Needs `SceNet_stub`/`SceNetCtl_stub` and `sceNetInit`. Melee's DebugNet path uses
UDP 18197; GDB uses TCP 1234.

## 4. On-screen debug screen

The cheapest debug UI is a `vita2d` text overlay: keep the last N lines in a ring
buffer, draw them in a corner, toggle with a button chord.

Play! ships one (`Source/ui_vita/debug_screen.cpp` + `font.c`). PrincessLog's
settings app contains a self-contained GXM debug screen with its own font blob
(`Tools/PrincessLog/src/NetLoggingMgr/settings_app/src/debugScreen.h`) — a useful
copy-paste if you have no font pipeline yet.

## 5. Logging plugins that exist in `Tools/`

| Tool | What it does |
|------|--------------|
| `PrincessLog` | `net_logging_mgr.skprx` + PC receiver + settings app: network logging for **any** app, including retail titles, without rebuilding it |
| `PSMLogUSB` | ring-buffer logger over USB |
| `ShipLog2.0` | alternative logger with a PC back end |

These matter when you cannot rebuild the thing you are debugging — e.g. watching
what a retail game does when you hook it.

## 6. Instrumentation that pays for itself

- zone timers around named regions, ring buffer, on-screen overlay
- draw-call counter, logged every 300 frames
- shader cache hits per tier (RAM / writable / packaged / compiled)
- memory: per-subsystem peak, asserted in dev builds
- I/O: bytes and milliseconds per request

## 7. Release builds strip all of it

Melee's rule: Release = `-O3 -DNDEBUG`, no logging, no debugger, no symbols,
diagnostics behind explicit build flags. Ship what you tested *in shape*, not just
in logic.

## 8. Sources

- `Other Ports/melee-Vita-vita-port/platforms/vita/vita_log.c` and `platforms/vita/README.md`
- `Other Emulators/play-master/Source/ui_vita/debug_screen.cpp`
- `soloader-template/source/utils/logger.c`
- `Tools/PrincessLog/`, `Tools/PSMLogUSB/`, `Tools/ShipLog2.0/`
