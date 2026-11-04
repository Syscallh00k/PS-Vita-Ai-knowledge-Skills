# 34 — Build, Install & Error Catalogue

Codes and linker messages you will actually see, and what each one means.

> Derived from [yannis100/vita-port-skill](https://github.com/yannis100/vita-port-skill)
> (`references/debugging.md` §2–3, MIT). See [CREDITS](../CREDITS.md).

---

## 1. Install / bubble errors

| Code | Meaning | Fix |
|------|---------|-----|
| `0x8010113D` | LiveArea PNGs not indexed, or wrong format/size | run `check_vpk.py` (upstream, see doc 39); fix with `pngquant` or its `make_placeholder_assets.py` |
| `0x8010111C` | `head.bin` missing | include it, or regenerate the package |
| `0x80101104` | corrupt `app.db` entry | rebuild the database / reinstall |
| `0x80010016` | directory limit reached | fewer files; pack assets into one archive |
| `C2-12828-1` | generic "app aborted or hung" | get a core dump |

Bubble missing or wrong:

- title ID must be **exactly 9 characters** and unique
- `template.xml` and the referenced images must exist
- unsafe builds need "Enable Unsafe Homebrew" in HENkaku settings
- rebuilding the database from Safe Mode sometimes fixes a stale entry

## 2. Runtime error codes worth memorising

| Code | Meaning | First response |
|------|---------|----------------|
| `0x8002710B` | NO_MEMORY | shrink `_newlib_heap_size_user`, or reinstall with `ATTRIBUTE2=12` |
| `0x805B0002` | GXM OUT_OF_MEMORY | lower texture/render-target sizes, drop MSAA |
| `0x80010002` | invalid argument / bad handle | often a NULL from a failed alloc used unchecked |
| `0x80101A04` | FIOS not found | wrong path or archive not mounted |
| `0x8002D0xx` | module load failure | `sceSysmoduleLoadModule` refused; check the module id |

Resolve the rest with `Tools/PSVita-error-code-resolver` (doc 15).

## 3. Link errors

| Message | Cause / fix |
|---------|-------------|
| `undefined reference to sce*` | add the matching `Sce*_stub` |
| `undefined reference to __cxa_*`, `operator delete` | add `-lstdc++` (vitaGL is C++) |
| `undefined reference to sceDmacMemcpy` (from `libvitaGL.a`) | add `SceKernelDmacMgr_stub` |
| `undefined reference to sceSharedFb*` (from `libvita2d.a`) | this SDK has **no `libSceSharedFb_stub.a`**; either define no-op stubs, or skip vita2d and use GXM/vitaGL directly |
| pthread symbols unresolved | use `-pthread`, not bare `-lpthread` |
| `vita-elf-create` relocation errors | link with `-Wl,-q` |
| `cannot find -lSceAppmgr_stub` | library names are **case sensitive**: `SceAppMgr_stub`, `vitaGL`, `OpenSLES` |
| vitaGL symbols missing | check order: `-lmathneon -lvitashark -lSceShaccCgExt -ltaihen_stub -lSceShaccCg_stub` |
| `sceGxm*` undefined from `libvitaGL.a` although `SceGxm_stub` is listed | static archives are **order sensitive** — `SceGxm_stub` must come *after* `vitaGL`. `SceGpuEs4_stub` / `libScePiglet_stub` are not needed for this (proven by relinking) |
| GCC 14/15 errors in old C (`implicit-function-declaration`, `int-conversion`, `incompatible-pointer-types`, `return-mismatch`) | fix the casts, or scope `-Wno-error=` to that file; `-std=gnu11` first |
| `vita-elf-create: Cannot allocate N bytes for SCE data at end of segment 0; segment 1 overlaps` | data segment lands on the next 64 KiB boundary and the tool appends ~5 KiB of SCE metadata; about 8% of text sizes fail. Add a `.rodata` pad (`const unsigned char pad[32*1024] = {1};`) and fail the build early with a gap check |
| `vita-elf-create: ... is not an ELF file` (size still changing) | a previous `ld` is still running (build piped into `head`/interrupted). Wait for `ld`, rebuild without the pipe |
| so-loader: unresolved symbols / wrong struct sizes | add to `dynlib.c` / `java.c`; wrap bionic-vs-newlib structs; check `__stack_chk_guard` |

## 4. The gap check worth automating

After link, before packaging:

```
gap = second_LOAD_vaddr - first_LOAD_vaddr - first_LOAD_memsz
```

`arm-vita-eabi-readelf -lW <elf>` prints the program headers. If the gap is smaller
than ~8 KiB, the packaging step is about to fail — fail the build yourself, with the
number, instead of leaving a confusing `vita-elf-create` message for the next person.

## 5. Boot-crash triage in one table

| Symptom | Check first |
|---------|-------------|
| crash on boot, no log | heap larger than free MAIN (heap init silently leaves an empty heap and every `malloc` returns ENOMEM) |
| same, second guess | main-thread stack too small — set `sceUserMainThreadStackSize` or run the game on a big thread |
| same, third guess | missing `-Wl,-q`, or the ELF never went through `vita-elf-create` |
| `UNSAFE` build refuses to start | unsafe homebrew not enabled |
| freezes 1–2 s on first use | runtime shader compilation; add a shader cache |
| animations frozen / fades stuck | `time()` has 1-second resolution; use `sceKernelGetProcessTimeWide()` |
