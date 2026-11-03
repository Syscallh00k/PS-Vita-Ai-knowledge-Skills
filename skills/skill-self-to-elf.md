---
name: self-to-elf
description: Identify and convert SELF/ELF containers on Vita and PS3, verify the parse with arithmetic identities, and know which tool does which direction.
when: You have a SELF/ELF file and need to know what it is, or need a disassemblable ELF.
---

# Skill: SELF → ELF

## Step 1 — Identify by magic

| First bytes | Container |
|-------------|-----------|
| `53 43 45 00` (`"SCE\0"`) | SCE SELF — Vita **or** PS3 |
| `7F 45 4C 46` | plain ELF |
| `50 53 46` | `param.sfo`, not code |
| `~PSP` | PSP EBOOT |
| high entropy, no magic | packed by the game, not the platform |

Vita and PS3 SELFs share the magic. Disambiguate by reading `version` and the ELF
header at `elf_offset` (Vita = ELF32 little-endian `e_machine` 40; PS3 = ELF64
big-endian `e_machine` 21).

## Step 2 — Read the header before converting

PS3-style SELF header is big-endian:

```
+0x04 u32 version
+0x32 u64 elf_offset          <- plaintext ELF header lives here
+0x3A u64 phdr_offset
+0x42 u64 shdr_offset
+0x12 u64 header length
+0x1A u64 elf file size
```

Vita SELF is little-endian with a similar layout (`psp2`-era `self.h` in
`Tools/vita-unmake-fself/src/self.h`).

**Key fact:** the ELF header at `elf_offset` is plaintext even when segments are
encrypted. You can always learn the target machine, entry point, and segment map
before decrypting anything.

## Step 3 — Verify with identities

```
header_len + elf_filesize          == file size
shdr_offset + e_shnum * e_shentsize == file size
elf_offset + sizeof(Ehdr)          == phdr_offset
phdr_offset + e_phnum * e_phentsize == section_info_offset
```

If any of these fail, you misread a field — fix the parse before proceeding.

## Step 4 — Convert

| Goal | Tool |
|------|------|
| SELF → ELF | `Tools/vita-unmake-fself` (`tool input.bin` → `input.bin.elf`) |
| ELF → SELF | `Tools/PSVita-ELF-builder` |
| inject ELF into an existing SELF | `Tools/elf_injector` |
| decrypt a running game's segments | `Tools/FAGDec` (kernel + user plugin pair) |
| decompile to pseudo-C | `Tools/vitadecompiler-mod` (`prxtool` based) |
| extract kernel bootloader/modules | `Tools/psp2-kbl-elf-extract`, `Tools/psp2-kernel-bootimage-extract` |

## Step 5 — Locate module info defensively

`Tools/psp2-kbl-elf-extract/src/module_info_parser.c` is the reference:

1. scan section headers for `.sceModuleInfo.rodata`
2. if absent and `e_type == ET_SCE_RELEXEC` (or `ET_SCE_EXEC` with an entry),
   use `phdrs[0].p_offset + e_entry`
3. otherwise use `phdrs[0].p_paddr`
4. sanity-check before trusting any offset

That ordering exists because the layout differs by module type. Guessing produces
garbage export tables and hours of confusion.

## Step 6 — Analyse

```
arm-vita-eabi-objdump -d input.elf | less
```

For imports/exports by name, feed the ELF to `Tools/THGN` with a library YAML
database (see skill `nid-mapping`).

## Never

- never patch ciphertext — a SELF is signed, and editing bytes produces a bricked
  copy
- never redistribute keys, key files, or decrypted retail binaries
- never skip the identity checks; a wrong offset reads as plausible garbage
