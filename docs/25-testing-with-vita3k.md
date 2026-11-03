# 25 — Testing with Vita3K

`Emulator/Vita3K.exe` runs VPKs on your PC. It is a fast filter, not a correctness
oracle.

---

## 1. What it is good for

| Catches | Why it works |
|---------|--------------|
| bad `param.sfo` / missing LiveArea assets | parses and mounts like the console |
| crashes before display init | same code path, real debugger attached |
| path assumptions (`app0:` vs `ux0:`) | emulates both filesystems |
| gross memory over-allocation | different limits, but obviously-wrong sizes fail |
| missing modules | module import resolution is emulated |

## 2. What it lies about

| Area | Why |
|------|-----|
| GXM behaviour | renders through Vulkan; SGX quirks do not exist |
| timing / shader compile cost | desktop CPU, no `libshacccg` |
| overclock | meaningless |
| memory ceiling | desktop has far more |
| texture swizzle mistakes | often still looks right |

So: if it fails in Vita3K, it is almost certainly your bug. If it passes in Vita3K,
you have learned very little about how it will run on hardware.

## 3. Config worth knowing

`Emulator/config.yml`:

```yaml
backend-renderer: Vulkan
resolution-multiplier: 1
texture-cache: true
async-pipeline-compilation: true
gdbstub: false          # turn on to attach gdb to the running app
log-active-shaders: false
log-uniforms: false
validation-layer: true
```

`gdbstub: true` is the reason to keep Vita3K installed even after hardware works —
it is the only comfortable way to single-step ARM Vita code.

`log-active-shaders` / `log-uniforms` tell you exactly what your renderer asked the
GPU to do, which is useful when a scene is black on hardware and fine in the
emulator (or vice versa).

## 4. Workflow

```
 build ──▶ Vita3K ──▶ hardware
             │            │
        fails┘            └fails
      logic bug         timing / memory / GPU / plugin
```

Run this split every time something breaks. It removes half the search space for
free.

## 5. Installing packages

Vita3K installs VPKs and can also install a folder title. For development the
fastest loop is still: FTP the `eboot.bin` to the console, and drop a copy into the
Vita3K app directory for the emulator.

## 6. Keep the symbol file

Whatever you test with, keep the matching `.elf`. When a dump or a core file arrives
weeks later, an unsymbolised crash log is a hex blob.

## 7. Sources

- `Emulator/config.yml` (this workspace's settings)
- `Emulator/Vita3K.exe`
- `Games/*.vpk` (known-good packages for validating the install path)
