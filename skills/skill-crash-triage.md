---
name: crash-triage
description: Work out why a Vita app died — read the symptom, decode the error, get a dump, and remove half the search space by testing in Vita3K first.
when: The app exits, hangs, or returns a strange code.
---

# Skill: Crash Triage

## Step 1 — Classify the death

| Behaviour | Class | Next step |
|-----------|-------|-----------|
| returns to LiveArea immediately | boot failure | step 3 (boot gates) |
| exits silently after some time | assert or abort | step 2 |
| hangs, app still "running" | deadlock / infinite wait | step 4 |
| freeze then system dialog | kernel panic | step 5 |
| runs, renders wrong | not a crash | skill `first-frame` |

## Step 2 — Stop silent exits

Override the handlers once, at the top of the project:

```c
void abort(void) {
    LogWrite(LOG_ERROR, "abort() called");
    sceKernelExitProcess(0);
    while (1) sceKernelDelayThread(1000000);
}

void __assert_func(const char* file, int line, const char* fn, const char* expr) {
    LogWrite(LOG_ERROR, "assert: %s at %s:%d (%s)", expr, file, line, fn);
    abort();
}
```

Five lines that convert "it just closes" into a filename and line number.

## Step 3 — Boot gates

```c
CHECK(module_loaded("kubridge"), "kubridge.skprx missing");
CHECK(file_exists("ux0:data/game/base.archive"), "data missing");
CHECK(file_exists("ur0:data/libshacccg.suprx"), "shader compiler missing");
CHECK(sceIoOpenWrite_ok("ux0:data/game/"), "data dir not writable");
```

A one-line on-screen message beats a three-frame-later crash.

## Step 4 — Deadlock checklist

| Check | Fix |
|-------|-----|
| is an IO completion dispatched on the IO thread? | dispatch on the game thread |
| is a lock held across a GPU call? | move the GPU call out |
| is the audio thread allocating? | preallocate |
| is teardown joining a thread owned by a callback? | detach or use a flag |
| is a "wait for ready" flag set after the wait? | reorder |

Add a watchdog: a low-priority thread that logs every 2 s with a frame counter. If
the counter stops, you know it is a hang, not a crash.

## Step 5 — Errors and dumps

- every negative return is a hint: `Tools/PSVita-error-code-resolver` maps
  `0x8xxxxxxx` to text using the firmware `error_table.bin`
- kernel dumps: `Tools/Kdumper` (dump) + `Tools/kdump-extract` (extract)
- physical memory: `Tools/physmem-dumper`
- core files: `vita-parse-core` against the matching `.elf`

**Keep the `.elf` for every build you put on the device.** Without it, a dump is a
hex blob.

## Step 6 — Halve the problem with Vita3K

Boot the same VPK in `Emulator/Vita3K.exe`:

| Result | Conclusion |
|--------|-----------|
| fails identically | logic or packaging bug — debug on PC with `gdbstub: true` |
| passes | timing, memory ceiling, GPU, or a missing plugin on hardware |

This one step is the highest-value thing in the whole document.

## Step 7 — The ten-minute checklist

1. log file exists and has content?
2. last line logged?
3. boot gates all pass?
4. works in Vita3K?
5. any negative return values logged?
6. free memory at crash point?
7. any change to shader count or texture count before it died?
8. did the last change touch lifecycle (launch, relaunch, exit)?
9. does it reproduce with a clean `ux0:data/<game>/`?
10. does it reproduce on a second boot without reinstalling?

## Never

- never chase a crash you cannot log
- never ship a build whose symbol file you threw away
- never "fix" a crash by adding a sleep
