# 41 — Kernel Debugging with GDB (kvdb)

Attaching a real debugger to a process on the console. Slow to set up, and the only
way to answer "what is it doing right now" when logging is not enough.

> Sources: `Other Emulators/kvdb-master` (David "Davee" Morgan, MIT),
> `Other Ports/melonDS-Vita-master` (unstripped ELF discipline).
> See [CREDITS](../CREDITS.md).

---

## 1. How the pieces fit

```
gdb (pc)  ──UART──▶  kvdb.skprx (kernel plugin)  ──▶  target process
             └─TCP─▶  vdbtcp companion plugin
```

`kvdb` provides the **kernel-space** half. It does not launch anything: you start the
application in debug mode from LiveArea/VitaShell first, then attach.

| Path | Needs |
|------|-------|
| UART | a hardware UART (devkit or a console with one wired) |
| TCP | the separate `vdbtcp` companion plugin |

Install by adding to `config.txt`:

```
*KERNEL
ux0:tai/kvdb.skprx
```

## 2. What it exposes to userland

`kvdb` publishes a kernel library as **syscall-able user functions** — the pattern
worth copying for any kernel feature you want userland to drive:

```yaml
kvdb:
  modules:
    vdb_kernel:
      functions:
        - vdb_serial_uart
        - vdb_serial_pipe
        - vdb_launch_debug
    vdb_user:
      syscall: true
      functions:
        - vdb_serial_uart
        - vdb_serial_pipe
        - vdb_send_serial_pipe
        - vdb_recv_serial_pipe
        - vdb_launch_debug
```

With `syscall: true`, a userland binary can call these directly, which is how the
transport is configured from inside the app being debugged:

```c
void vdb_serial_uart(void);                              /* use UART  */
void vdb_serial_pipe(void);                              /* use pipes */
int  vdb_send_serial_pipe(const char *data, size_t size);
int  vdb_recv_serial_pipe(char *data, size_t max_size, int timeout);
int  vdb_launch_debug(const char *executable);
```

## 3. Known limitations (alpha software)

From the project's own README:

- **one debug session per boot** — after a disconnect, reboot
- **memory faults are not correctly handled** — a SIGSEGV may not stop where you
  expect, or at all
- treat it as a breakpoint/step/inspect tool, not a reliable post-mortem tool

Given those, the practical workflow is: use it while the process is alive, and use
core dumps (doc 15) for crashes.

## 4. What you must have on hand

| Item | Why |
|------|-----|
| `.elf` with symbols, per build | attach and symbolise without it is pointless |
| the exact `.velf` you installed | line numbers must match the running code |
| a build with `-g` | `Release` with `-O3` reorders and inlines everything |
| source tree at the same commit | gdb lists files by path |

melonDS-Vita's Makefile encodes the discipline that makes this possible:

```make
%.velf: %.elf
	cp $< $<.unstripped.elf      # keep symbols before stripping
	$(PREFIX)-strip -g $<
	vita-elf-create $< $@
```

Every build that touches hardware leaves a matching `.unstripped.elf` behind. Do the
same, name it with the version or git hash, and keep it.

## 5. When GDB is worth the setup cost

| Question | Better tool |
|----------|-------------|
| "does this function get called" | a log line |
| "what is this value" | a log line |
| "which of these 400 draw calls breaks state" | render trace (doc 14) |
| "why did it hang *here*" | **GDB** |
| "what does the call stack look like" | **GDB** |
| "did the crash corrupt the heap before dying" | core dump (doc 15) |

Do not reach for GDB first. Reach for it when logging has already narrowed the
problem to a location and you need state, not history.

## 6. Emulator alternative

`Emulator/Vita3K.exe` with `gdbstub: true` gives the same GDB experience on a desktop,
with faster iteration and no console risk. Use it for anything that reproduces there;
keep kvdb for the bugs that only exist on hardware (doc 25 explains why the split
matters).
