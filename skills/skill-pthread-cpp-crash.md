---
name: pthread-cpp-crash
description: Diagnose a C++ Vita port that exits on its first std::thread or corrupts the heap when threads exist — the libstdc++ pthread_cancel probe, and the link fix.
when: A C++ app dies instantly when starting a thread, throws system_error, or corrupts memory only under threading.
---

# Skill: pthread / C++ Crash

## Step 1 — Recognise the signature

| Symptom | Suspicion |
|---------|-----------|
| exits (or returns to LiveArea) exactly when a `std::thread` starts | this trap |
| `std::system_error`, `EPERM`, or "operation not permitted" in the log | this trap |
| heap corruption that appears only once a second thread exists | this trap (non-atomic `shared_ptr`) |
| audio-thread + game-thread race that "cannot happen" | this trap |
| crash deep in unrelated code, only under load | maybe this trap; check anyway |

Why: **VitaSDK libstdc++ uses the address of `pthread_cancel` as its
thread-availability probe.** If that symbol was never linked in, the address is 0 →
std::thread concludes threading is unavailable → throws, and `shared_ptr` refcounting
drops to non-atomic.

## Step 2 — Check the symbol

```bash
arm-vita-eabi-nm build/app | grep pthread_cancel
```

- `U` (undefined) → **you have the bug**
- `T` (defined) → this is not your problem, look elsewhere

## Step 3 — Fix the link

```cmake
target_link_libraries(app
    -Wl,--whole-archive pthread -Wl,--no-whole-archive
    ...)
```

Not `-lpthread`. Bare `-l` lets the linker drop archive members that nothing
references directly — and `pthread_cancel` is referenced only through an address
probe.

Recheck: `nm` should now show it defined. Rebuild fully (not incremental).

## Step 4 — Give threads real stacks

While you are here:

| Where | Size |
|-------|------|
| main thread | `sceUserMainThreadStackSize` = 8 MiB |
| worker threads | **256 KiB minimum**, not 32 KiB |
| pthread default | too small for C++ + newlib |

newlib polling alone can reserve ~9 KiB per thread. 32 KiB stacks "work" until they do
not.

## Step 5 — Verify with a deliberate thread test

```c
#include <thread>
#include <memory>

auto SelfTest() noexcept -> bool {
    std::shared_ptr<int> p = std::make_shared<int>(42);
    bool ran = false;
    std::thread t([&] { ran = (p.use_count() > 0); });
    t.join();
    return ran;
}
```

If this throws or crashes, stop and fix the link before debugging anything else. If it
passes, threading is functional.

## Step 6 — Still crashing?

Then it is a real bug, not the trap. Go to:

- doc 35 (static fault analysis) for address → named defect
- doc 15 (crash diagnostics) for dumps and error codes
- `skill-crash-triage` for the general checklist

## Never

- never patch `pthread_cancel` with a dummy function — the *address* is the probe, and
  a stub that does nothing is still wrong when a thread is actually cancelled
- never raise stack sizes to hide a recursion bug
- never assume "it links" means the thread machinery is present
