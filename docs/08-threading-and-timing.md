# 08 — Threading & Timing

Four A9 cores. Threads are easy; priorities and handoffs are not.

---

## 1. Two threading APIs

| API | Header | Use |
|-----|--------|-----|
| newlib `pthread` | `pthread.h` | portable code, third-party libraries |
| `sceKernel*` threads | `psp2/kernel/threadmgr.h` | precise priorities, tiny stacks, kernel-side work |

`pthread` is the default. Link it with `--whole-archive` or the glue is dropped:

```cmake
-Wl,--whole-archive pthread -Wl,--no-whole-archive
```

**Not optional for C++.** VitaSDK libstdc++ probes threading by taking the address of
`pthread_cancel`; if that archive member gets dropped, `std::thread` throws
`system_error` and `shared_ptr` silently falls back to non-atomic refcounting. Both
failures are documented, with the fix, in
[docs/48-pthread-and-libstdcpp-traps.md](48-pthread-and-libstdcpp-traps.md).

## 2. Priorities

Higher numeric value = higher priority on Vita kernel threads. A working example:

```
#define DVD_THREAD_PRIORITY 0x10000100
#define DVD_THREAD_STACK    (64 * 1024)
```

Rules:

- audio thread outranks the render thread (crackle is worse than a dropped frame)
- IO thread should be *below* audio but must not starve — give it its own core
- never spin in a high-priority thread; `sceKernelDelayThread` is your friend

## 3. The standard four-thread layout

```
 main      : scene traversal, draw submission
 stream    : archive reads → decompress → upload queue
 upload    : format conversion, swizzle, GPU-visible buffer writes
 audio     : mix + SceAudioOut submit
```

Handoff pattern: double buffer + atomic index swap. No mutex held across a read of
more than a few microseconds.

## 4. Clocks

```c
scePowerSetArmClockFrequency(444);
scePowerSetBusClockFrequency(222);
scePowerSetGpuClockFrequency(222);
scePowerSetGpuXbarClockFrequency(166);
```

Lock these **first**, at boot, before any timing measurement — otherwise profiling
numbers drift depending on what ran before. PSVshell can push higher; assume the
user may have it and never depend on it.

## 5. Timing primitives

| Need | Call |
|------|------|
| monotonic, µs-class | `sceKernelGetSystemTimeWide()` |
| wall clock | `sceRtcGetCurrentTick`, `sceRtcGetCurrentClockLocalTime` |
| sleep | `sceKernelDelayThread(us)` |
| frame pacing | busy-wait on the display sync, not `sleep` — sleep granularity is coarse |
| vblank events | `sceDisplayWaitVblankStart`, `sceDisplayGetVcount` |

Frame pacing pattern used by emulator frontends:

```c
static void PacingWaitTick(SceUInt64* deadline, SceUInt64 interval) noexcept {
    SceUInt64 now = (SceUInt64) sceKernelGetSystemTimeWide();
    if (*deadline <= now) { *deadline = now; }
    while ((SceUInt64) sceKernelGetSystemTimeWide() < *deadline) { /* spin */ }
    *deadline += interval;
}
```

Spin-waiting the last few hundred microseconds beats `sleep` for stable 60 Hz.

## 6. Deadlock patterns specific to this platform

| Pattern | Symptom | Fix |
|---------|---------|-----|
| IO completion dispatched from the IO thread | freezes after seconds | dispatch on the game thread |
| Audio callback allocating memory | random hangs | preallocate all voices |
| `pthread_join` in a teardown path that a callback owns | app never exits | detach, or use a flag + `sceKernelExitProcess` |
| lock held while calling into the GPU | stutter spike | move GPU calls out of the critical section |

## 7. Sources

- `Other Ports/melee-Vita-vita-port/platforms/vita/game/dvd.c`, `audio.c`
- `Other Emulators/PCSX_AwO-master/frontend/vita/standalone.c` (pacing)
- `soloader-template/source/reimpl/pthr.c` (Android pthread reimplementation)
- `Sdk/include/psp2/power.h`, `kernel/threadmgr.h`
