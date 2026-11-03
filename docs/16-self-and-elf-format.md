# 16 — SELF & ELF Format

Vita ships `SELF`. Toolchains produce `ELF`. Understanding both is required for
tooling, reverse engineering, and understanding your own build.

---

## 1. SELF header (Vita)

```
+0x00 char[4]  magic "SCE\0"
+0x04 u32      version
+0x08 u16      sdk_type / flags
+0x0A u16      header_type
+0x0C u32      metadata_offset
+0x10 u64      header_len
+0x18 u64      elf_filesize
+0x20 u64      self_offset
+0x28 u64      appinfo_offset
+0x30 u64      elf_offset        <- plaintext ELF header copy
+0x38 u64      phdr_offset
+0x40 u64      shdr_offset
...
```

Reading `elf_offset` gives you the ELF header without decrypting anything. That
single fact makes SELF triage fast.

Vita-specific ELF `e_type` values that appear in modules:

| Value | Meaning |
|-------|---------|
| `ET_SCE_EXEC` | executable, entry point |
| `ET_SCE_RELEXEC` | relocatable executable (most modules) |
| `ET_SCE_REL` | relocatable object |

Tools in `Tools/` handle the conversion:

| Tool | Direction | Note |
|------|-----------|------|
| `vita-unmake-fself` | SELF → ELF | reads the header, extracts segments, writes `<input>.elf` |
| `PSVita-ELF-builder` | ELF → SELF | bundles a runme batch (`_RUNME.BAT`) |
| `elf_injector` (`vita-elf-inject`) | inject ELF into SELF | produces a patched loadable |
| `FAGDec` | decrypt a **running** game's SELF segments | kernel + user plugin pair |

## 2. sceModuleInfo

Every Vita module carries a `SceModuleInfo` structure describing name, version,
type, and its export/import tables. `Tools/psp2-kbl-elf-extract` locates it
robustly, which is a good reference implementation:

1. walk section headers looking for `.sceModuleInfo.rodata`
2. if absent, fall back to `phdrs[0].p_offset + e_entry` (relexec) or
   `phdrs[0].p_paddr` (others)
3. sanity-check offsets before trusting them

Copy that defensive order — module info layout varies by type, and guessing produces
garbage export tables.

## 3. ELF for analysis

Once you have an ELF:

```
e_machine: 40 (EM_ARM) for Vita
e_type:    ET_SCE_RELEXEC / ET_SCE_EXEC / ET_SCE_REL
```

What you can do with it:

| Task | Tool |
|------|------|
| Disassemble | `arm-vita-eabi-objdump -d` |
| Extract import/export tables | `THGN`, `vita-import` YAML |
| Decompile to pseudo-C | `Tools/vitadecompiler-mod` (wraps `prxtool`) |
| Patch and repack | `elf_injector`, `PSVita-ELF-builder` |

`vitadecompiler-mod` is TheFloW's decompiler modified by dots_tb/CelesteBlue — it
emits C-like output with resolved imports, which is how several ports began.

## 4. Non-Vita containers, for contrast

The same skills apply to PS3 games, with different numbers:

| Platform | Magic | Notes |
|----------|-------|-------|
| Vita SELF | `"SCE\0"` | little-endian ELF32 payload |
| PS3 SELF | `"SCE\0"` | big-endian ELF64 / PPC64 payload, encrypted segments |
| PS3 ELF | `7F 45 4C 46 02 02` | ELF64 big-endian, `e_machine` 21 |

A PS3 `EBOOT.BIN` and a Vita `eboot.bin` both start with `SCE\0`. Check the version
field and the ELF `e_machine` before assuming which platform you are looking at.

## 5. Sources

- `Tools/vita-unmake-fself/src/main.c` + `self.h`
- `Tools/psp2-kbl-elf-extract/src/module_info_parser.c` (the fallback chain)
- `Tools/FAGDec/src/app/dump.c` (running-process dump)
- `Tools/vitadecompiler-mod/`, `Tools/PSVita-ELF-builder/`, `Tools/elf_injector/`
- `docs/26-ps3-and-ue3-target-notes.md` for the PS3 side
