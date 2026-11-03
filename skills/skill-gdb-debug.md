---
name: gdb-debug
description: Attach a real debugger to a Vita process with kvdb — setup, prerequisites, what it can and cannot do, and when to reach for it instead of logs.
when: Logging has narrowed a bug to a location and you need live state, stacks, or stepping.
---

# Skill: GDB Debug

## Step 1 — Ask whether GDB is the right tool

| Question | Tool |
|----------|------|
| "is this called / what is this value" | a log line |
| "which draw breaks state" | render trace |
| "why is it hanging *here*" | **GDB** |
| "what is the call stack" | **GDB** |
| "what did the heap look like when it died" | core dump |

If logging can answer it, use logging. GDB costs a boot cycle per session.

## Step 2 — Prerequisites

- `kvdb.skprx` in `config.txt` under `*KERNEL` (UART), plus `vdbtcp` for TCP
- the app started **in debug mode** — kvdb does not launch targets
- a build with `-g`, and the matching **unstripped `.elf`** kept from that build
- the source tree at the same commit

## Step 3 — Keep symbols from the start

```make
%.velf: %.elf
	cp $< $<.unstripped.elf
	$(PREFIX)-strip -g $<
	vita-elf-create $< $@
```

Do this in every build script (melonDS-Vita does). A build without its `.elf` is
undebuggable forever.

## Step 4 — Know the limits before you trust it

kvdb's own README (alpha software):

- **one debug session per boot** — disconnect means reboot
- **memory faults are not correctly handled** — a SIGSEGV may not stop where you
  expect
- not a post-mortem tool: use core dumps for crashes

## Step 5 — Session shape

1. boot the console, start the app in debug mode
2. connect gdb to the transport (UART, or TCP via `vdbtcp`)
3. load symbols: `file build/<proj>.elf`, `set sysroot` if you staged libraries
4. breakpoint by file:line or symbol, then continue
5. inspect: `bt`, `info registers`, `x/…`, `p …`

Log what you learn into the issue/commit before you lose the session — one session
per boot is a hard budget.

## Step 6 — The cheaper alternative

`Emulator/Vita3K.exe` with `gdbstub: true` gives the same GDB experience on the
desktop. Use it for anything that reproduces there. Keep kvdb for hardware-only bugs
(timing, memory ceiling, missing plugin, GPU).

## Never

- never attach to a build whose symbols you did not keep
- never debug an `-O3`-only build — inline and reorder makes stepping meaningless
- never treat a kvdb memory-fault stop as trustworthy
