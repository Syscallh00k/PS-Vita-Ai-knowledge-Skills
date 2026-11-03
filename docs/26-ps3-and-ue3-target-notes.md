# 26 — PS3 / UE3 Target Notes

Measured notes from a real PS3 title (`Batman: Arkham Asylum`) kept here because the
*forensics* generalise to any PS3-era UE3 game.

---

## 1. SELF header (PS3)

Big-endian fields:

```
+0x00 char[4]  "SCE\0"
+0x04 u32      version (2 or 3)
+0x08 u16      flags
+0x0A u16      key revision
+0x0C u16      header type
+0x0E u32      metadata offset
+0x12 u64      header length
+0x1A u64      elf file size
+0x22 u64      self offset
+0x2A u64      appinfo offset
+0x32 u64      elf offset         <- plaintext ELF header lives here
+0x3A u64      phdr offset
+0x42 u64      shdr offset
```

Arkham's `EBOOT.BIN` (25,992,848 bytes):

| Field | Value |
|-------|-------|
| version | 2 |
| header length | 0x980 |
| elf file size | 0x18C9510 |
| elf offset | 0x90 |
| phdr offset | 0xD0 |
| section info offset | 0x290 |
| control info offset / size | 0x3C0 / 0x70 |

Verification identities that prove the parse (worth doing on any SELF):

```
header_len + elf_filesize          == file size
shdr_offset + e_shnum * 64         == file size
elf_offset + sizeof(Elf64_Ehdr)    == phdr_offset
phdr_offset + e_phnum * 56         == section_info_offset
```

## 2. Inner ELF

| Field | Value |
|-------|-------|
| `e_ident` | ELF64, big-endian |
| `e_type` | 2 (`ET_EXEC`) |
| `e_machine` | 21 (`EM_PPC64`) |
| entry | `0x01839928` |
| program headers | 8 × 56 bytes |
| sections | 30 × 64 bytes |

| # | type | vaddr | filesz | memsz |
|---|------|-------|--------|-------|
| 0 | `PT_LOAD` | 0x10000 | 0x17263C8 | 0x17263C8 |
| 1 | `PT_LOAD` | 0x1740000 | 0x1968F8 | 0x3C1510 |
| 5 | `PT_TLS` | 0x18A600C | 0x4 | 0x31C |
| 6 | `0x60000001` | 0x1736380 | 0x20 | 0x20 |
| 7 | `0x60000002` | 0x17363A0 | 0x28 | 0x28 |

`0x60000001`/`0x60000002` are Cell process-param and auxv program headers.

Scale: **24.2 MB of code + 4 MB of data**. That is the whole game — no separate
script payload to reuse, because console UE3 cooks strip UnrealScript into native
code.

## 3. Content

| Item | Measurement |
|------|-------------|
| packages | 2,490 `.XXX` (3,075.5 MB), 97 `.STR` (416.3 MB), 1 `.TFC` (324.8 MB), 6 `.BIN` |
| total cooked | 4,005,267,935 bytes (3.73 GiB) |
| movies | 78 `.BIK` (Bink 1), 2,913.9 MB |
| package tag | `0x9E2A83C1` — UE3, every file |
| alignment | every `.XXX`/`.TFC`/`.BIN` is a multiple of 32,768 B; `.STR` are not |
| texture cache | `TEXTURES.TFC` shares the UE3 tag, then `78 9C` (zlib) at 0x18 |

Package header observations (diff two files to derive, never assume):

| Offset | Observed |
|--------|----------|
| 0x00 | tag `9E 2A 83 C1` |
| 0x04 | `00 15` constant u16 |
| 0x06 | `02 40` constant u16 |
| 0x08 | varies per file — size/offset class |
| 0x0C | `00 00 00 05` then `"None\0"` — first name is always `None` |

Names are **not** stock UE3 `FString`. Measured encoding:

```
u8  length (includes NUL)
u8  chars[length]
3 bytes bookkeeping
```

verified on `08 "Admin_A\0"`, `0E "Admin_A_PhysX\0"`, `0C "Admin_A_CH4\0"`.

The name run in `Admin.xxx` is the level package list (`Admin_A`,
`Admin_A_PhysX`, `Admin_A_CH567_Grapple`, …) — i.e. the persistent-level map.

Two `.BIN` files are worth naming: `GLOBALSHADERCACHE-PS3.BIN` (264 KB — the game's
own shader permutation cache, the fastest route to a material taxonomy) and five
`COALESCED_*.BIN` localisation string tables.

## 4. Why this target is not a "port" in the usual sense

| Constraint | Consequence |
|------------|-------------|
| PPC64 binary, no source, no decomp | source-port and `.so`-load routes are closed |
| 24 MB of native game logic | gameplay must be re-authored, not recovered |
| RSX shader microcode only | material *classes* get rebuilt, not shaders translated 1:1 |
| 6.6 GB raw | content must be rebuilt at ~half size before shipping |
| 2.9 GB of Bink | transcode to H.264 (no Bink decoder exists for Vita) |

Emulation is not an option: a Cell PPU + 7 SPUs needs far more than a 444 MHz
Cortex-A9. Static recompilation of 24 MB of PPC64 is a multi-year compiler project
before the first level boots.

The realistic shape is a **framework reimplementation**: consume cooked content,
re-author gameplay, phase it so each stage is useful on its own.

## 5. The general reusable part

For any PS3-era game:

1. parse the SELF, confirm the ELF header, verify with the identities in §1
2. decrypt if needed (a `scetool`-class tool with a key file)
3. enumerate imports — every `cell*` / `sys*` symbol is a future shim
4. map the graphics path first (anything reaching `cellGcm*`)
5. parse containers before objects; validate against an independent tool
6. compute the storage budget before promising anything

## 6. Sources

- `Arkham Asylum/Game Binarys - ps3/PS3_GAME/USRDIR/EBOOT.BIN`
- `.../BMGAME/COOKEDPS3/*` (headers, sizes, alignment)
- `.../BMGAME/MOVIES/*.BIK`, `.../PS3TOC.TXT`
- `docs/15-crash-and-error-diagnostics.md`, `docs/16-self-and-elf-format.md`
