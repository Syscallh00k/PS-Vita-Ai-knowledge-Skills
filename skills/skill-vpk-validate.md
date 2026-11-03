---
name: vpk-validate
description: Run the pre-install checks that catch "installs but no bubble" and 0x8010113D — title ID, param.sfo, SELF magic, indexed PNGs, template.xml, linker gap.
when: You built a VPK and it does not install, does not appear, or the console reports a code.
---

# Skill: VPK Validate

## Step 1 — Run the checker

```bash
python3 check_vpk.py build/mygame.vpk      # upstream script, see doc 39
```

Exit 0 = no errors. It verifies:

| Check | Failure looks like |
|-------|--------------------|
| `eboot.bin` starts with `"SCE\0"` | installs, exits immediately |
| `param.sfo` parses | install fails silently |
| `TITLE_ID` exactly 9 characters | bubble missing |
| `APP_VER` matches `NN.NN` | metadata rejected |
| `icon0.png` 128×128, indexed, no alpha | blank bubble |
| `bg0.png` 840×500, `startup.png` 280×158 | stretched or missing art |
| `template.xml` well-formed | broken LiveArea page |

The script is not vendored here — it lives in
[yannis100/vita-port-skill](https://github.com/yannis100/vita-port-skill) under
`skills/vita-port/scripts/check_vpk.py` (MIT). Doc 39 describes what it verifies.

## Step 2 — Fix the most common three

1. **PNGs not indexed** → `pngquant`, or generate valid ones with the upstream
   `make_placeholder_assets.py` (doc 39)
2. **Title ID wrong length or duplicated** → `XXXXYYYYY`, 9 characters, unique
3. **`eboot.bin` is an ELF, not a SELF** → `vita_create_self` did not run

## Step 3 — Check the linker gap before blaming packaging

```
vita-elf-create: Cannot allocate N bytes for SCE data at end of segment 0; segment 1 overlaps
```

The data segment landed on a 64 KiB boundary and the tool appends ~5 KiB of metadata
behind the text segment.

```bash
arm-vita-eabi-readelf -lW build/<proj> | head -20
# gap = 2nd LOAD vaddr - 1st LOAD vaddr - 1st LOAD memsz
```

If the gap is under ~8 KiB, add padding:

```c
const unsigned char pad[32 * 1024] = { 1 };
```

and fail the build early with the gap printed, so the next person sees a number
instead of a mysterious packaging error.

## Step 4 — Map the console's error code

| Code | Meaning |
|------|---------|
| `0x8010113D` | LiveArea PNG format/colour type |
| `0x8010111C` | `head.bin` missing |
| `0x80101104` | corrupt `app.db` entry |
| `0x80010016` | directory limit |
| `C2-12828-1` | app aborted/hung — get a core dump |
| `0x8002710B` | out of memory (heap too large, or need `ATTRIBUTE2=12`) |
| `0x805B0002` | GXM out of memory |

Full table in doc 34; unknown codes via `Tools/PSVita-error-code-resolver`.

## Step 5 — Install path check

- VitaShell from `ux0:/vpk/`, Cross to install, accept the extended-permissions
  prompt for unsafe builds
- unsafe homebrew must be enabled in HENKaku settings
- for iteration, FTP `eboot.bin` into `ux0:/app/<TITLEID>/` instead of reinstalling

## Step 6 — Gate before every release

- [ ] `check_vpk.py` exit 0
- [ ] title ID unique on a device that already has your test builds removed
- [ ] `param.sfo` version matches the release tag
- [ ] LiveArea images present and the right size
- [ ] the eboot is the binary you tested (hash it)

That last one is not paranoia — a stale build directory has shipped before.
