---
name: https-downloads
description: Add working HTTPS downloads to a Vita app with libcurl and mbedTLS, because SceHttp/SceSsl cannot handshake with modern hosts.
when: A port needs to fetch content over HTTPS from the device.
---

# Skill: HTTPS Downloads

## Step 1 — Accept the premise

`SceHttp`/`SceSsl` **cannot complete a TLS handshake with current hosts**. This is
measured, not theoretical (StepMania Vita port, build 112).

So: ship your own stack. libcurl + mbedTLS, statically linked.

## Step 2 — Bundle a CA store

```
app0:/cacert.pem                     default bundle
ux0:data/<app>/cacert.pem            user override (expired bundle, custom pin)
```

Pinning the bundle in `app0:` makes verification deterministic regardless of what the
console's system store contains.

## Step 3 — Network init

```c
sceSysmoduleLoadModule(SCE_SYSMODULE_NET);
sceNetInit(&pool_size);

CURL* curl = curl_easy_init();
curl_easy_setopt(curl, CURLOPT_URL, url);
curl_easy_setopt(curl, CURLOPT_CAINFO, "app0:/cacert.pem");
curl_easy_setopt(curl, CURLOPT_NOSIGNAL, 1L);        /* required, no signals */
curl_easy_setopt(curl, CURLOPT_WRITEFUNCTION, OnWrite);
curl_easy_setopt(curl, CURLOPT_XFERINFOFUNCTION, OnProgress);
```

`CURLOPT_NOSIGNAL` is mandatory; the default relies on SIGALRM and will hang or abort.

## Step 4 — Policy, not just calls

| Rule | Why |
|------|-----|
| verification **on** by default | a project that disables it by default has no credibility |
| one explicit verification-off retry, logged | devices with a wrong clock see valid certs as expired |
| retry transient failures on a **fresh** connection | half-closed sockets look alive |
| bounded retries, then fail loudly | endless retry is a hang with extra steps |
| delete partial files | truncated assets surface later as "corrupt game data" |

## Step 5 — UI

- own screen, drawn with vita2d/FreeType, not inside the game renderer
- progress from the curl write/progress callback, not file-size polling
- **cancel button** — a download UI without cancel is unusable
- install into the data directory only after completion + verification

## Step 6 — Errors go on screen

Use `SceMsgDialog` with the actual message. Network failures that only appear in a log
generate bug reports containing no information.

## Step 7 — Verify on hardware

Vita3K's network does not prove device behaviour. Check:

- [ ] plain HTTP works
- [ ] HTTPS verifies against the bundled CA
- [ ] wrong console clock → clear error, escape hatch works
- [ ] cancel mid-download leaves no partial file
- [ ] resume/second attempt succeeds after an intentional failure
- [ ] memory does not grow across repeated downloads

## Never

- never parse HTML to find a download URL — hit an API or a static file
- never download on the render thread
- never trust a response without a size/checksum check
