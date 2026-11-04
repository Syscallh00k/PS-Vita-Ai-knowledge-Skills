# 49 — Networking and TLS That Actually Works

`SceHttp`/`SceSsl` are not enough for the modern web. This is the measured workaround,
from a shipping port that had to make downloads work on device.

> Source: `new imports/stepmania-psvita-main/README.md` (build 112 technical notes),
> `new imports/godot4-psvita-main` (LiveArea launch params).
> See [CREDITS](../CREDITS.md).

---

## 1. The finding

> "`SceHttp`/`SceSsl` cannot complete a TLS handshake with current hosts."

So any port that needs to download from a real HTTPS endpoint has to bring its own
stack. The StepMania Vita port ships one, and its configuration is worth copying:

| Piece | Choice |
|-------|--------|
| transport | **libcurl** |
| TLS | **mbedTLS** |
| CA store | bundled `app0:/cacert.pem`, overridable by `ux0:data/<app>/cacert.pem` |
| verification | **stays on** |
| fallback | a verification-off retry, only for devices whose clock makes certs look expired |
| retries | transient errors retried over a **fresh connection** |
| cleanup | half-written files removed |

That is a complete, honest policy. Copy the shape, not just the libraries.

## 2. Why each decision is there

**Bundle your own CA store.** The Vita's clock and its system store are both
unreliable in ways you cannot control. A pinned `cacert.pem` in `app0:` makes
verification deterministic; the `ux0:` override lets a user fix an expired bundle
without a rebuild.

**Keep verification on, and provide an explicit escape hatch.** Turning TLS
verification off by default is how a project gets a reputation. Making it a
documented, user-visible fallback is how it stays honest.

**Retry on a fresh connection, not the same one.** Mobile-class TLS stacks leave
half-closed sockets that look alive. A new connection costs milliseconds and avoids a
class of hangs.

**Delete partial files.** Interrupted downloads leave truncated assets that later
fail as "corrupt game data", sent to you as a bug report about your loader.

## 3. Clock skew is a real failure mode

A Vita whose RTC has drifted sees valid certificates as not-yet-valid or expired.
The practical trio:

1. always verify
2. if verification fails with a date-related error, report it clearly (that tells the
   user to fix the clock rather than blame you)
3. offer the verification-off retry as an explicit, logged action

## 4. Link set

```
-lcurl -lssl -lcrypto   (or -lmbedtls -lmbedx509 -lmbedcrypto)
-lSceNet_stub -lSceNetCtl_stub
```

Plus `sceSysmoduleLoadModule(SCE_SYSMODULE_NET)` and `sceNetInit` before any socket
call, and `SceHttp`/`SceSsl` only if you stay on plain HTTP or your own endpoints
still negotiate.

## 5. LiveArea action tiles — the underused feature

StepMania's song-pack browser only opens from a **LiveArea tile**, never from a normal
launch. That is the clean solution to "this feature should not be reachable
mid-game".

How it works:

```
LiveArea template.xml   → an action frame (282x108 in their layout, "frame9")
                        → launches the app with a param, e.g. psla:packs
app side               → sceAppUtilReceiveAppEvent(&event)
                        → sceAppUtilAppEventParseLiveArea(&event, buffer)
                        → strstr(buffer, "psla:packs") → open the browser
```

Three things this buys you:

- a second entry point with no in-game UI cost
- a place for network-y or long-running features that should not disturb gameplay
- the user can launch straight into it from the home screen

The same mechanism is how Android-loader ports open a configurator
(`app0:configurator.bin`) — see doc 12 §3.

## 6. Downloading with a progress UI

Practical notes from the same port:

- run the downloader on its own screen, drawn with `vita2d` + FreeType, not inside the
  game's renderer
- report progress from the curl write callback, not by file size polling
- support **cancel** — a download UI without cancel is a hang with extra steps
- install into the app's data directory only after the file is complete and verified

## 7. Error reporting

Use the system dialog rather than a silent failure or a log nobody reads:

```c
/* SceMsgDialog with the actual message */
```

Network operations are the one place where "it failed quietly" is guaranteed to
produce a bug report with no information in it.

## 8. Checklist for adding network features to a port

- [ ] `SceSysmodule` NET loaded, `sceNetInit` called, errors logged
- [ ] does the feature need HTTPS? if yes: libcurl + mbedTLS, bundled CA
- [ ] verification on by default, escape hatch documented
- [ ] retry policy: fresh connection, bounded count, user-visible outcome
- [ ] partial files deleted on failure and on cancel
- [ ] progress + cancel UI
- [ ] errors surface in a system dialog
- [ ] works with a device whose clock is wrong
- [ ] tested on hardware, not just Vita3K (its network stack is not the device's)
