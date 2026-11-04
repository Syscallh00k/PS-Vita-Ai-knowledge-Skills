---
name: engine-as-so
description: Split a large engine into a PIC .so plus a small loader that resolves its imports — when it pays off, what extra files it needs, and how to verify it at package time.
when: The engine is large, changes constantly, or you want content swaps without rebuilding the app.
---

# Skill: Engine as Shared Library

## Step 1 — Is it worth the split

| Situation | Split? |
|-----------|--------|
| engine will change for months, huge libc surface | yes |
| you want to swap content sets without touching code | yes |
| you need symbols on device for GDB | yes (ship stripped, keep full) |
| small app, a dozen imports | no — link statically |

## Step 2 — Build the engine as PIC

Godot's Vita path as a concrete example:

```sh
python3 -m SCons platform=vita arch=arm32 target=template_release \
  optimize=size vita_shared=yes vita_gl_root=/abs/path/vitagl -j6
```

Result: `bin/libgodot.so` — static engine objects linked in, external libraries left as
unresolved imports **on purpose**.

## Step 3 — Write the loader

You need these files; there is no way around them:

| File | Job |
|------|-----|
| `so_util.c` | load, relocate, resolve (same code Android loaders use) |
| `default_dynlib.c` | one table row per imported symbol |
| `tls_stubs.c` | **thread-local storage** — engines use `thread_local` heavily and newlib needs help here |
| `dl_stubs.c` | `dlopen` / `dlsym` / `dlclose` |
| `sdk_shims.c` | small libc differences |
| `boot_splash.c` | something on screen during load (a black screen reads as a crash) |

Skip `tls_stubs.c` and you get corruption with no error message.

## Step 4 — Verify at package time, not boot time

The packager must:

1. validate the ELF (format, relocations, entry point)
2. **check every import against `default_dynlib.c`** and abort with the symbol name
3. `strip --strip-unneeded` the shipped copy, re-validate the stripped ELF, keep the
   unstripped original
4. write a manifest with hashes of runtime, renderer library, loader and assets
5. have the loader print build id + hashes on the first lines of its log

Step 2 is the whole value of the pattern: a missing import becomes a build error
instead of a device crash.

## Step 5 — Memory

The loader reads the whole `.so` before mapping segments, so a fat `.so` costs
temporary memory equal to its size. Hence the strip step. Budget:

```
.so on disk + full read buffer + mapped segments + your data
```

If the `.so` is 40 MB, that is the price of admission.

## Step 6 — Test order

1. `.so` loads and prints its manifest line
2. engine `main` runs, exits cleanly
3. display comes up (clear colour)
4. one shader path works
5. content loads

Do not test content before step 4.

## Step 7 — Diagnosis

| Symptom | Cause |
|---------|-------|
| loader exits with a symbol name | import missing from the dynlib table |
| crashes inside engine init | TLS shims missing or wrong |
| heap corruption immediately | TLS, or a library that was linked statically *and* imported |
| works, but only after a second launch | you are not resetting state between runs |
| black screen before anything logs | no boot splash, and the load takes seconds |

## Never

- never ship a `.so` you have not import-verified
- never link libraries both statically into the `.so` and as imports
- never assume the loader can stay unchanged — treat `default_dynlib.c` as source
