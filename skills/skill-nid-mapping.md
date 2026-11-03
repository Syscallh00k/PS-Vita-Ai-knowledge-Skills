---
name: nid-mapping
description: Resolve Vita NIDs to names, build import/export stubs from YAML, and pick the right hooking method for the job.
when: A binary shows numeric imports, a link fails on an undefined symbol, or you need to hook by NID.
---

# Skill: NID Mapping

## Step 1 — Understand the indirection

```
module (SceGxm) → library (SceGxm) → function (NID 0xA8E2C4B8 = sceGxmBeginScene)
```

A NID is a 32-bit hash of the function signature. Stub libraries contain NIDs, not
names. That means:

- a link error is a **missing stub**, not a missing header
- a runtime `0x8002xxxx` is a **module not loaded**
- you cannot hook by name at ABI level — only by NID, offset, or address

## Step 2 — Link errors

```
undefined reference to `sceGxmBeginScene'
```

Find the library:

```powershell
Get-ChildItem $env:VITASDK\arm-vita-eabi\lib\libSceGxm*
```

Then add `SceGxm_stub` to `target_link_libraries`. Same pattern for every `Sce*`
module; the stub name matches the module name.

## Step 3 — NID → name (the useful direction)

`Tools/THGN` does this end to end:

1. parse the ELF's import/export tables (`src/import/vita-import-parse.c`)
2. look each NID up in a bundled YAML database (`findNameByNid`)
3. print or emit hook entries

Result: `0x6D9F1C4E` becomes `sceKernelGetProcessId`, which is the difference between
guessing and knowing.

## Step 4 — Build a stub from YAML

VitaSDK's `vita-import` tool turns a YAML description into an import/export table
embedded in your module. Every plugin in `Tools/` that has an `exports.yml` works
this way:

```yaml
base_hooker:
  attributes: 0
  version:
    major: 1
    minor: 1
  main:
    start: module_start
    stop: module_stop
```

Use this when you are *exporting* something (a service other code calls) rather than
importing.

## Step 5 — Choose the hooking method

| Method | API | Stability |
|--------|-----|-----------|
| export hook by NID | `taiHookFunctionImport` / `...ForKernel` | good across builds |
| export hook by offset | `taiHookFunctionOffset` | per-build |
| direct patch | `kuKernelCpuUnrestrictedMemcpy` | per-build **and** per-address-layout |

Decision rule:

- shipping to users → hook by NID, and degrade gracefully if the hook fails
- single known build you control → offset is fine and simpler
- patching instructions → only with a byte signature check first

## Step 6 — Verify a hook worked

```c
int uid = taiHookFunctionOffset(&ref, modid, 0, offset, 1, MyFunc);
if (uid < 0) {
    LogWrite(LOG_ERROR, "hook failed: 0x%08X (offset mismatch?)", uid);
}
```

Always log the UID. A silent failed hook looks exactly like "my patch does nothing".

## Step 7 — Release what you take

`taiHookRelease` in `module_stop`, every time. A leaked hook corrupts the import
table for subsequent launches, which manifests as unrelated crashes elsewhere.

## Never

- never hardcode a NID you have not verified against the target's own imports
- never assume NIDs are identical across firmware versions for obscure APIs
- never ship a plugin that fails silently
