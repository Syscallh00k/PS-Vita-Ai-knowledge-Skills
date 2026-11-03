---
name: streaming-io
description: Load large or many assets from Vita storage without hitching — one IO thread, bounded queue, completions on the game thread, one archive.
when: A port loads a lot of data, or the game stutters whenever something new appears.
---

# Skill: Streaming I/O

## Step 1 — Measure the source of truth

Before designing, find out:

| Question | How |
|----------|-----|
| how many files does a level need? | count them |
| what is the largest single file? | list by size |
| how many small files (< 64 KB)? | if thousands, one archive is mandatory |
| what is the read speed? | time a 64 MB sequential read |

## Step 2 — One archive

Thousands of small files destroy load times. Build one indexed container
(doc 24 §7) with O(1) lookup and sequential-friendly layout. Mount it via FIOS, or
read it with your own reader.

## Step 3 — One IO thread, bounded queue

```
 main ──request──▶ [queue cap 32] ──▶ io thread (SCE_O_NOWAIT)
  ▲                                          │
  └────────── completions polled here ◀───────┘
```

- bound the queue: dropping a request and drawing a lower LOD beats stalling
- one thread only — card seeks are the bottleneck, not CPU
- `SCE_O_NOWAIT` for non-blocking handles

## Step 4 — Dispatch completions on the game thread

The classic hang: an IO callback fires on the IO thread and touches game state that
the game thread is also touching. Funnel completions into a lock-free list and drain
them from the frame loop (Melee's `melee_vita_platform_poll()`).

## Step 5 — Prefetch by design

| Trigger | Action |
|---------|--------|
| level load | read the whole cell's index + small assets |
| approach a region | start the next cell 2–3 seconds ahead |
| texture needed now, not loaded | use a low-res proxy, swap later |
| shader needed now, not compiled | fallback program, log a miss |

## Step 6 — Verify

- log bytes and milliseconds per request; a 2 ms read and an 80 ms read should not
  be the same code path
- assert peak staging memory
- test with the card nearly full (free-space fragmentation is real)

## Diagnosis

| Symptom | Cause |
|---------|-------|
| hitch exactly when a model appears | synchronous read on the render thread |
| hitch every few seconds | many small opens |
| freeze | completion dispatched on the wrong thread |
| slow only on real hardware | file count; the emulator has a fast host FS |
| corruption after suspend | buffered handle not closed/reopened |

## Never

- never `fopen` per asset during gameplay
- never block the render thread on a file read
- never assume `ux0:data/` exists — create your directory tree at first run
