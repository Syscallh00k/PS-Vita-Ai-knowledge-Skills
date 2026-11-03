# 35 — Static Fault Analysis

Turning a crash report into a named defect without reproducing it. This is the
highest-leverage debugging skill on this platform, because crashes on hardware are
expensive to reproduce and cheap to read.

> Derived from [yannis100/vita-port-skill](https://github.com/yannis100/vita-port-skill)
> (`references/debugging.md` §5–6, MIT). See [CREDITS](../CREDITS.md).

---

## 1. Resolve the address first — always

Before calling anything "an emulator fault":

```bash
arm-vita-eabi-readelf -S build/<proj>.velf | grep .text   # is it even in your image?
arm-vita-eabi-addr2line -f -C -e build/<proj> 0x810e4cb0   # unstripped add_executable output
```

A "Trace/breakpoint trap" that resolved to `_kill_r` was `abort()` all along — two
sessions were spent blaming the emulator. **Resolve, then reason.**

## 2. Check the arithmetic, and check that it is code

- symbolised `Foo::bar` +838 means `address + 838`; redo the addition
- compare the offset against the symbol's size:
  `arm-vita-eabi-nm -C -S --defined-only build/<proj> | grep '<function>'`.
  An offset larger than the function size is not in that function; it resolved to a
  different symbol.
- a "faulting address" that lands in a literal pool (`.word 0x8240c400`) is data and
  can never have been the faulting instruction. `objdump` the neighbourhood.

## 3. Disassemble the whole function

```bash
arm-vita-eabi-objdump -d -C --start-address=<fn> --stop-address=<fn+size> build/<proj>
```

A symbolised "invalid read" is often a compiler-folded consequence rather than the
line you expect.

## 4. Recognise the null-dereference signature

```
movs r0, #0
ldr  r0, [r0, #0x58]
```

That is not a corrupted pointer at runtime — it is the compiler proving a pointer is
NULL and still emitting the access, so the fault address is literally `0x58`.

Procedure:

1. find `0x58` in the class's member list to name the field
2. search the source for the member access
3. look at *why* the pointer was NULL — very often an accessor that returns a
   constant and never reads `this`:

```cpp
u32 getType() const { return 'NJNT'; }   // never dereferences
...
else if (pJoint->getType() == 'NJNT') { // "null check" that always passes
    J3DMaterial* m = pJoint->getMesh(); // folds to ldr r0, [r0, #0x58]
}
```

Any method of that shape is not a null check. Test `p != NULL` explicitly.

## 5. Inherited or introduced?

Compare against the reference tree before "fixing" upstream code:

```bash
git diff HEAD -- <file>
```

If the diff is empty and the reference decomp has identical code, the bug is
inherited and the fix belongs in a documented local patch, not in a silent edit.

## 6. Verify the fix in the codegen

Rebuild, re-disassemble, confirm the `movs r0, #0` / `ldr r0, [r0, #N]` pair is
gone and the field access now goes through a live register. A grep of the source
cannot prove the optimiser stopped folding.

## 7. Say what the fix does not prove

If the original log lost the faulting instruction, the honest claim is "the one
reachable null dereference in this area is closed", not "the reported crash is
explained". Record both in the commit.

## 8. Making a silent `abort()` name itself

`abort()` is a strong global in newlib, so the linker can redirect every call site
without touching the ported code:

```cmake
add_link_options(-Wl,--wrap=abort -Wl,--wrap=__assert_func -Wl,-Map=${CMAKE_BINARY_DIR}/app.map)
```

```c
extern void __real_abort(void) __attribute__((noreturn));

void __wrap_abort(void) {
    LogWrite("abort from %s at %p", g_context, __builtin_return_address(0));  /* flush! */
    __real_abort();
}
```

Details that cost time to learn:

- `__builtin_return_address(0)` is valid in the wrapper; **anything above 0 is
  unsafe** without frame pointers. Log level 0 only.
- **A `Release` build defines `NDEBUG`**, so `assert()` compiles to nothing and
  `__assert_func` is never referenced (`nm -u build/<proj> | grep -c assert` → 0).
  Wrapping it is still free and correct, but do not claim assertions are now
  captured.
- Emit `-Wl,-Map=` **before** you need it — it is often absent by default — and
  convert it into an embedded address→name table so the fatal path needs no
  filesystem read and no allocation. ~36k symbols ≈ 1.8 MB of `.rodata`.
- VitaSDK's newlib has **no `SA_SIGINFO`, no `sa_sigaction`, no
  `siginfo_t::si_addr`**. Use plain `signal(SIGABRT, handler)`.
- Trigger it deliberately through a file in `ux0:data` rather than an environment
  variable: emulator harnesses usually do not forward env into the guest but do
  share the data directory.
