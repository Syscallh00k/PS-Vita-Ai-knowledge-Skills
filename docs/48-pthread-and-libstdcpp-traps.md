# 48 — pthread and libstdc++ Traps on VitaSDK

Two settings decide whether a C++ port runs or dies in ways that look like memory
corruption. Both are one link flag away.

> Sources: `new imports/DeadBabiesX-psvita-port-main/CRASHFIX.md`,
> `new imports/flare-engine-master/Makefile.vita`,
> `new imports/stepmania-psvita-main`. See [CREDITS](../CREDITS.md).

---

## 1. The `pthread_cancel` trap

Measured on a real port, with disassembly to prove it:

> VitaSDK **libstdc++ uses the address of `pthread_cancel` as its thread-availability
> probe.**

The failure chain, exactly as documented:

```
pthread_cancel left unresolved (weak)      → probe address == 0
        ↓
std::thread considers threading unavailable → throws system_error / EPERM
        ↓
top-level exception handler exits the process   (looks like an instant crash)
        ↓
and, silently: shared_ptr refcounting falls back to NON-ATOMIC
        ↓
audio thread + game thread race on shared state → heap corruption, random crashes
```

Fixes, both required:

1. **Link the full pthread library**, so `pthread_cancel` is defined:
   `-Wl,--whole-archive pthread -Wl,--no-whole-archive` (or `-pthread`, which on
   VitaSDK pulls more of the library than bare `-lpthread`)
2. **Give threads real stacks** — that port went from 32 KiB to 256 KiB because the
   C++ worker plus newlib's polling code needed the headroom (poll alone reserved
   ~9 KiB in their toolchain)

Symptom summary, so you recognise it:

| What you see | What it probably is |
|--------------|---------------------|
| app closes the moment it starts a `std::thread` | the probe above |
| `std::system_error` / `EPERM` in a log | same |
| random heap corruption once threads exist | non-atomic `shared_ptr` from the same probe |
| crash deep in unrelated code, only when audio is playing | shared state raced by a second thread |

## 2. Always link pthread this way

```cmake
target_link_libraries(app
    -Wl,--whole-archive pthread -Wl,--no-whole-archive
    ...)
```

Not `-lpthread`. Archive members that are only referenced indirectly (probes,
weak hooks, `std::thread` internals) get dropped by a plain `-l`, and dropping them
is exactly what triggers the behaviour above.

## 3. Stack sizes, in practice

| Where | Value | Source |
|-------|-------|--------|
| main thread | set `sceUserMainThreadStackSize` (8 MiB is a sane default) | doc 01 |
| worker threads | 256 KiB minimum for C++ + newlib | DeadBabiesX crash fix |
| default pthread stack | too small for C++ code that recurses or uses iostream | — |

If a port crashes "in deep code" only on device, raise thread stacks before you
chase the bug.

## 4. Other link/language flags that appear in every working port

Consensus from flare-engine, StepMania, DeadBabiesX, melonDS:

```
-marm                       ARM encoding; faster on A9 than Thumb-2 in hot code
-fno-exceptions             where the engine tolerates it (not for std::thread users)
-fno-optimize-sibling-calls keeps stacks walkable for dumps/GDB; melonDS uses it
-ffast-math -O2/-Ofast      normal
-fno-strict-aliasing        for engine code that type-puns
-fno-lto                    flare disables LTO deliberately
```

`-fno-optimize-sibling-calls` is the one people skip and then cannot read a crash
dump. It is worth the few percent.

## 5. The probe-check recipe

If you are unsure whether the trap applies to your build:

1. `arm-vita-eabi-nm build/app | grep pthread_cancel` — undefined symbol (`U`) means
   the probe will read zero
2. `arm-vita-eabi-objdump -d build/app | grep -A3 pthread_cancel` — look at how the
   address is taken
3. Fix the link, rebuild, re-check that it is now `T` (defined)
4. Only then re-test threading behaviour

## 6. Why this matters more than it looks

The trap converts a link-flag omission into two unrelated-looking failures
(instant exit *and* heap corruption) which appear at different times and different
places. Ports have chased the "random gameplay crash" for weeks while the cause was
one missing archive member.

If a C++ Vita port misbehaves at all, check this first.
