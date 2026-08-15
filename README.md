# Marie LRU

**Multi-graded Adaptive Reclaim & Independent Eviction**

Marie is an out-of-tree, third-pillar LRU implementation for the Linux kernel, sitting alongside Legacy LRU and MGLRU rather than replacing either.  It composes three established ideas — working-set protection ([le9uo](https://github.com/firelzrd/le9uo)), proportional anon/file aging under a generational ring ([Re-swappiness](https://github.com/firelzrd/re-swappiness)), and async swap-out offload ([kcompressd-unofficial](https://github.com/firelzrd/kcompressd-unofficial)) — onto a redesigned core: a flat per-PFN state array with no linked lists or staging queues, a single global generation ring per memory type without forced oldest-gen drain, and a SIMD-accelerated PTE walker driven by rmap-to-walker bloom-filter feedback. Two mechanisms then fall out of that core rather than being bolted onto it: reclaim accounting derived from state-byte transitions, and an optional defragmenter that reuses the same array and generations in place of stock compaction.

Marie is **desktop / global-only**: it runs a single node-global aging clock and reclaim scan and keeps *zero* per-memcg/per-lruvec reclaim state. cgroup *charging* (`memory.current`, PSI) stays on the stock path, but cgroup memory limits are not enforced through Marie's reclaim. This trades multi-tenant isolation — irrelevant on a single-user desktop, where Marie's near-OOM stability subsumes the stability purpose of cgroup limits and `oom_score_adj` covers OOM targeting — for a dramatically simpler, more robust core. (It is therefore unsuited to containers/servers/Android that depend on per-cgroup reclaim isolation.)

Marie is selected once at boot behind a static key (`lru_marie_enabled()`), so a kernel built with `CONFIG_LRU_MARIE=y` but booted with `lru_marie=0` behaves exactly as MGLRU / Legacy LRU would. There is no runtime on/off toggle — the key is set before any folio is tracked, so the `enabled` sysfs node is read-only.

---

## Background

Linux's page-reclaim LRU has gone through two main eras.

**Legacy LRU** (the upstream default before MGLRU, and still the fallback) maintains per-memcg active/inactive list pairs.  Folios are promoted on `PG_referenced` and demoted on scan-tail age-out.  The model is small and cheap, but the binary active/inactive boundary is coarse — within each bucket there is no further ordering, so when the inactive list isn't large enough to span the working set, refault-driven re-promotion turns into a thrash treadmill.

**MGLRU** (`lru_gen`, mainlined in 6.1) replaced the two-bucket model with a generational ring, actively harvested by a PTE walker rather than relying solely on rmap-side `PG_referenced`.  This delivered a large improvement on workloads where the active/inactive cut sliced through the working set, and remains the right baseline for most general-purpose use.

Marie is a third design point.  The axes where MGLRU's choices turn into trade-offs under specific workloads — and where Marie picks the other end:

Legend — ✅ behaves as expected · ⚠️ works but with a structural trade-off · ❌ not provided / cannot deliver the property

| Aspect                | Legacy LRU                                                                | MGLRU                                                                                                       | Marie                                                                                                       |
| :-------------------- | :------------------------------------------------------------------------ | :---------------------------------------------------------------------------------------------------------- | :---------------------------------------------------------------------------------------------------------- |
| Aging granularity     | ⚠️ binary active / inactive only                                           | ✅ 4-gen ring (`MAX_NR_GENS=4`); a full ring triggers a forced `inc_min_seq` oldest-gen drain                 | ✅ 8-gen ring (`MARIE_PFN_NR_GENS=8`); advance skips when the ring is full, no forced drain                  |
| anon/file iteration   | ✅ per-type lists; `vm.swappiness` works as a proportional anon:file ratio | ❌ shared `max_seq` across anon/file reduces `vm.swappiness` to a per-generation hint                        | ✅ per-type `seq` / `min_seq` / ring; restores `vm.swappiness` as a proportional ratio                       |
| Hot-signal harvest    | ⚠️ rmap-driven `PG_referenced` only; no active walker                      | ✅ PTE walker + walker-internal PMD bloom filter + rmap look-around cascading `PG_referenced`                | ✅ Vectorized PTE walker + rmap-fed PMD bloom filter + rmap look-around that promotes straight to the head gen and cascades no `PG_referenced` |
| Compression stall     | ❌ kswapd blocks inside `zswap_store`                                      | ❌ kswapd blocks inside `zswap_store`                                                                        | ✅ offloaded to per-node `kcompressd` kthread                                                                 |
| Clean-file floor      | ❌ none                                                                    | ❌ none                                                                                                      | ✅ `clean_min_ratio` hard floor against thrashing                                                            |
| Reclaim-livelock OOM  | ❌ retries until `MAX_RECLAIM_RETRIES`                                     | ❌ retries until `MAX_RECLAIM_RETRIES`                                                                       | ✅ three independent closures: the `clean_min_ratio` floor makes reclaim stop cleanly so the stock no-progress OOM fires promptly; `zone_reclaimable_pages()` discounts anon by the compressed store's own RAM cost so that test is not lied to on zram; plus a swap-write-failure fast-path |
| RAM-backed swap (zram) | ❌ anon always *looks* reclaimable → OOM may never arm                     | ❌ same                                                                                                      | ✅ reclaimable anon is reported net of the zspages the eviction will consume, measured from live counters — no ratio, no tuning constant |
| Defragmentation       | ⚠️ stock compaction                                                        | ⚠️ stock compaction                                                                                          | ✅ optional (`CONFIG_LRU_MARIE_DEFRAG`) — reuses the per-PFN array + gens to score and clear near-empty pageblocks inside kcompactd |
| cgroup reclaim / isolation | ✅ per-memcg active/inactive lists; `memory.max` enforced via targeted reclaim | ✅ per-memcg gen state; `memory.max` enforced via targeted reclaim | ❌ desktop/global-only — single node-global scan, no per-cgroup reclaim isolation (`memory.max` not enforced through Marie; cgroup *charging* / `memory.current` / PSI stay accurate) |

These are deliberate choices on MGLRU's side: fewer state bits per folio, fewer fast-path branches, a single iteration counter shared between memory types, a self-contained walker that owns its own feedback.  Marie's complementary design accepts the resulting state cost in exchange for proportional `vm.swappiness` control under a generational ring, hard working-set protection, and a hot-signal pipeline that pulls the rmap into the loop instead of letting it cascade `PG_referenced` onto neighbouring folios.

---

## Lineage

```
              le9uo           Re-swappiness        kcompressd-unofficial
                │                   │                       │
        clean_min_ratio         per-type            kcompressd kthread +
        (hard file floor)       seq / min_seq       FIFO write-back
                │                   │                       │
                └─────────┬─────────┴───────────────────────┘
                          │
                       Marie LRU
                          │
                  (own pipeline core:
                   flat per-PFN state array,
                   global per-type gen ring
                   w/o forced oldest-gen drain,
                   SIMD PTE walker,
                   rmap-fed bloom)
```

Marie keeps the **goals** of its predecessors but doesn't reuse their implementations verbatim — it re-derives each idea inside its own LRU core so the surrounding state (gen ring, lock topology, scan order) is internally consistent.

- From **le9uo**: hard working-set protection (`clean_min_ratio`).  Like le9uo, Marie enforces the floor on the single path that actually reclaims and then lets the kernel's stock no-progress OOM path fire — once clean file is at the floor and anon is unreclaimable, reclaim returns no progress and the OOM killer fires promptly, with no separate early-OOM watermark needed (this is why le9uo converges at any floor size).  `anon_min_ratio` and `clean_low_ratio` from le9uo are intentionally **not** ported — under Marie the former is largely redundant with the effective-swappiness policy once `kcompressd` brings anon refault sub-millisecond, and the latter is subsumed by the gen ring, which already orders cold file below warm file without a second level.
- From **Re-swappiness**: per-type aging so swappiness acts as a proportional anon:file scan ratio under a generational ring (the property Legacy LRU already has and MGLRU loses by sharing `max_seq`).  Re-swappiness ports this onto MGLRU's structure; Marie owns its own per-type aging in node-global per-type statics (`marie_head_gen[type]` plus independent per-type gen rings) and does not depend on MGLRU being per-type.
- From **kcompressd-unofficial**: per-node async compression FIFO so kswapd doesn't block in `zswap_store` / `__swap_writepage`.  Marie carries this over under the original `kcompressd` name — same per-node FIFO design, one `kcompressd<node>` kthread per node, signed −100..+100 depth knob, default +24.

---

## Architecture

```
                 fault path
                      │
                      ▼
             ┌────────────────────────────┐
             │ marie_state[pfn]           │  flat 1-byte per-PFN array
             │  (TRACKED, ISOLATED,       │  (no linked lists, no staging)
             │   TYPE, ZONE, GEN)         │
             └──────────┬─────────────────┘
                        ▲
   rmap look-around     │   promote to head gen
   promote (cmpxchg)    │   on young PTE (SIMD scan)
           │            │           │
           ▼            ▼           ▼
   ┌────────────────────────────────────────┐
   │        L1 / L2 tracking bitmaps        │
   │  marie_track_bm[type][zone][gen]       │
   └────────────────────┬───────────────────┘
                        │
                        ▼ (L2-pruned __ffs/blsr isolate)
              ┌───────────────────┐
              │ shrink_folio_list │
              └─────────┬─────────┘
                        │
                  anon  │  file
                        ▼
             ┌──────────────────────┐
             │ kcompressd kthread   │
             │  per-node FIFO       │
             │  zswap_store /       │
             │  __swap_writepage    │
             └──────────────────────┘
```

- **Flat per-PFN state array (`marie_state`)**: Marie abandons linked lists entirely. Every folio's state is encoded in a single byte (TRACKED, ISOLATED, TYPE, ZONE, GEN) within a flat array allocated once at boot. This eliminates allocation in the fast paths and per-CPU staging queues.
- **L1/L2 tracking bitmaps**: To efficiently scan the flat array, Marie maintains one summarized L2 bitmap per `(type, zone, gen)` plane — 64 planes in all — so the isolate scanner skips whole empty PFN ranges in a single cycle. The granularity is chosen at boot: a plane targets ~512 bits (`MARIE_L2_TARGET_BITS`, "a handful of cache lines"), capped so one L2 bit never covers more than 8192 pages (~32 MiB at 4 KiB), which is what bounds the per-hit L1 skim on large machines. Boot-tunable via `lru_marie_max_l2_pages_per_bit=`.
- **Independent per-type rings**: Aging is tracked by `marie_head_gen[type]` cycling counters. Anon and file have fully independent rings, restoring swappiness as a proportional scan ratio without forcing oldest-gen drains.
- **ISOLATED as an ownership token**: bit 6 marks a PFN as exclusively owned by an in-flight isolate/putback. Every other GEN mutator — walker promote-on-access, `MADV_COLD`, `MADV_FREE`, defrag restamp — must check it first and decline. This is also what makes the reclaim accounting exactly-once (see below).
- Hot-signal pickup is split across rmap and the walker. MGLRU also uses a bloom filter, but it's walker-internal — a self-feedback to skip PMDs that the walker itself did not touch on the previous pass. Marie's bloom is cross-component: rmap is the producer (the PTL-bounded look-around flags PMDs that just took a young hit), the walker is the consumer (it pays the SIMD scan + promotion cost only on bloom-hit PMDs). This lets a hot signal observed by rmap influence the walker within the same reclaim window, not the next one.

---

## Design

### Reclaim pipeline: Flat per-PFN state array

Marie tracks a folio's lifecycle entirely via a single byte per PFN in `marie_state[pfn]`. All 8 bits are used, ordered MSB → LSB by hierarchy rather than convenience:

| Bits | Field | Meaning |
| :--- | :--- | :--- |
| 7 | **TRACKED** | 1 = owned by Marie; 0 = the rest of the byte means nothing. The single source of truth — no `folio->flags` bit is used. |
| 6 | **ISOLATED** | 1 = exclusively owned by an in-flight isolate/putback. Every other GEN mutator checks this and declines. |
| 5 | **TYPE** | 1 = file LRU, 0 = anon LRU |
| 4..3 | **ZONE** | `folio_zonenum` (DMA / DMA32 / NORMAL / MOVABLE) |
| 2..0 | **GEN** | relative position 0..7 in the cycling ring (0 = oldest, head = `marie_head_gen[type]`) |

TRACKED and ISOLATED are gates — existence, then ownership — that must be read before anything else in the byte is meaningful; TYPE and ZONE are near-immutable identity attributes of the physical page; GEN is the one field that changes on nearly every access or reclaim pass, so it sits where every relocation's shift/mask lands. Untracked PFNs read as 0.

There is no hotness *field*: a folio observed young is promoted straight to the head gen by a single CAS on this byte. Earlier revisions carried a saturating tier counter here; narrowing it from 4 levels to 1 bit is what bought GEN its third bit, and retiring that last bit is what freed bit 6 for ISOLATED. (`marie_state_inc_tier()` keeps its historical name.)

Because state is purely array-based, there are no `list_del` or `swap-pop` operations during eviction or installation. Zeroing the byte instantly drops tracking. To isolate folios, Marie uses an **L2-pruned `__ffs`/`blsr` walk** combined with **Two-Stage Software Prefetching**:
- **L2-pruning**: Scanning the `(type, zone, gen)` L2 bitmap for the target bucket skips empty PFN ranges (up to ~32 MiB per bit) in 1 CPU cycle.
- **Hardware Extraction**: Surviving L2 blocks are resolved to precise PFNs in near O(1) time using `__ffs` (Find First Set) and `blsr` (reset lowest set bit) hardware instructions over the L1 bitmap.
- **Strategic Prefetch**: A "Cache-line cursor" tracks boundaries to issue exactly one prefetch per cache-line. It fires `prefetcht2` (DRAM → L3) far ahead of the processing cursor, and `prefetcht0` (L3 → L1) slightly ahead, hiding the DRAM latency wall without thrashing the L1 cache.

### Accounting: derived from state transitions, not from call sites

Marie's LRU size counters (`marie_nr_folios`, the per-`(gen, type)` occupancy, `__update_lru_size`) are never adjusted by whichever code path happens to be running. Each is derived from the *committed* transition of the state byte:

```
d = pred(new_byte) - pred(old_byte)      where pred(b) = TRACKED(b) && !ISOLATED(b)
```

`pred()` means "this PFN currently holds an outstanding +1". A path that publishes a byte it has already published contributes `d = 0`; a path that debits a folio some other path already debited contributes `d = 0`. Double-counting is therefore not a bug to be detected but an unrepresentable state, and ISOLATED doubles as the exactly-once token for the uncharge backstop that runs at `folio_put()` time. This replaced a family of hand-written per-site adjustments in which one miscounting branch was enough to drive the LRU counters negative and cause premature OOM.

### Gen-ring advance policy: skip on full, no forced drain

MGLRU's ring has `MAX_NR_GENS=4`.  When it fills, aging calls `try_to_inc_min_seq()` to push the oldest gen forward before cutting a new head.  If that oldest gen holds folios that won't move — clean file pages under a working-set policy, mlocked anon, etc. — aging still demands forward progress, so the same untouchable folios get rewalked over and over: the *forced `inc_min_seq` treadmill*.

Marie uses an 8-gen cap (`MARIE_PFN_NR_GENS`, from 3 GEN bits) tracked by `marie_head_gen[type]`. Head-generation advance is *drain-wait gated*: `marie_try_advance_head_mlv()` will not advance if the next slot in the ring still contains folios of that type.

Practical effect: an oldest gen full of `clean_min_ratio`-protected folios is left alone.  Aging stops growing new heads until reclaim drains the existing tail (or `clean_min_ratio` diverts the type selector to anon).  The treadmill never starts because nothing in Marie demands aging-side progress regardless of consumer state.

**Why 8 gens.** Spending the byte's third bit on the ring rather than on a hotness field lengthens a folio's head→oldest descent — its time-domain reclaim grace — giving the walker's hot-signal harvest more passes to re-promote a *warm* folio, one re-accessed on a period comparable to that descent, before it reaches reclaim range. In a warm-set A/B this cut warm-folio refaults ~40% versus a 4-gen ring with a 4-level hotness field. A 4-gen / 2-level control did not reproduce the gain, which isolates it to the gen count rather than to the promotion threshold — so the hotness field was retired outright and promote-on-access now goes straight to the head gen.

**Gen size is automatic.** A new head is cut once the current one has taken `marie_gen_growth_live[type]` pages' worth of installs. That threshold is computed, not configured — there is no knob:

```
live[anon] = max(occ/NR_GENS, (memtotal - occ)/NR_GENS, memtotal/256)
live[file] = max(occ/NR_GENS,  reserve/NR_GENS,         memtotal/256)
```

`occ/NR_GENS` is the scale-invariant steady state: the ring spans the type's working set, so the oldest gen holds ~1/8 of it and a folio ages over roughly one set-turnover of installs (counted in pages, not folios, so THP does not skew it). The anon-only slack term keeps the ring spanning the *whole* anon set while anon is still filling below half of RAM, then hands off to `occ/NR_GENS` in exactly the under-pressure regime where fine cold-folio selection matters. The file-only `reserve` term (the `clean_min_ratio` reserve) keeps stratifying file across all gens even when file has been squeezed down to its protected floor. `memtotal/256` is an absolute floor for the near-empty / high-churn corner. Recomputed only at a head advance and on a `clean_min_ratio` write, so the per-install hot path is one `READ_ONCE`.

### Independent anon/file pipelines

Marie keeps **fully independent** per-type state — each of ANON and FILE has its own `marie_head_gen[type]` cycling counter, install cadence and tracking bitmaps. There is no shared iteration counter between the two types, so swappiness works as a proportional anon:file scan ratio — the behaviour Legacy LRU has always had, recovered inside a generational ring. (Note that with the default `low_swappiness_mode=1` the *effective* swappiness is clamped to at most 1, which selects the FILE-then-ANON regime rather than the proportional controller; see [Recommended Configuration](#recommended-configuration).)

### Hot-signal harvest

- **SIMD PTE walker** (per-pgdat).  On x86-64, `arch_initcall` picks the widest available SIMD instruction set (AVX-512F > AVX2 > SSE2) — capped by the `simd_max` knob — and flips a static branch so the walker's young-bit extraction has no indirect call. Each scan reads 512 64-bit PTEs at once and reduces them to a 512-bit young-bit mask via vector ops, entirely avoiding per-PTE branching. Both `simd` (on/off) and `simd_max` (ISA cap) are runtime-tunable and boot-seedable. ARM64 and other arches use a scalar fallback for now.
- **Per-PMD FPU bracket, promotion drained outside the locks**.  `kernel_fpu_begin/end` wraps only the SIMD scan of a single PMD and never spans PMDs or bloom-miss stretches, so there is no batching knob to tune: save/restore is skipped entirely for kthreads (kswapd) and costs at most one save per kernel entry otherwise, which makes per-PMD bracketing effectively free. The heavy per-young-PTE work is then split in two — the `ptep_test_and_clear_young` aging runs under the PTL because it must, collecting young PFNs into a per-CPU worklist, and the head-gen promotions are drained *after* the PTL is dropped. The dominant cost therefore runs lock-free, preempt-enabled, and outside the FPU bracket.
- **rmap look-around**.  Called from `folio_referenced_one()`, this scans up to `BITS_PER_LONG` PTEs around the target folio's PMD under the existing PTL.  Unlike MGLRU's look-around it does **not** set `PG_referenced` on neighbouring folios — that cascade is the main source of reclaim starvation under fault-heavy workloads.  Instead it feeds the walker via a bloom filter (below).
- **rmap-fed PMD bloom feedback**.  MGLRU's bloom filter is walker-internal: the walker remembers which PMDs it touched last pass and uses that to skip cold PMDs next pass.  Marie's bloom is cross-component — rmap is the producer ("this PMD had a young PTE on its target"), the walker is the consumer ("only scan PMDs the bloom flagged").  The filter is double-buffered and swapped at the end of each walker pass, so each pass acts on the rmap signal accumulated during the previous reclaim window.
- **Pressure-adaptive cadence**.  The walker re-evaluates its scan interval (`walker_interval_{critical,low,normal,idle}_ms`) on each pass.  Defaults: `HZ/30`, `HZ/10`, `HZ/4`, `HZ` — all tunable.

### Pressure resilience

- **`clean_min_ratio` (hard working-set floor).**  Marie withholds file reclaim — on *every* reclaim path (the per-PFN pick driver and the legacy drain that follows it in `shrink_lruvec`) — when clean file pages would otherwise be evicted below the configured percentage of node RAM, diverting reclaim to anon instead. Only clean file counts toward the floor: dirty pages, which cannot be reclaimed without writeback, are subtracted so they cannot satisfy the floor on unreclaimable pages. Equivalent in spirit to le9uo's knob of the same name.  Default 10%; set to 0 to disable.

  The pick driver selects the reclaim type via a strict priority cascade: OOM victims and anon-unreclaimable conditions force FILE-only (no point swapping if no slots exist or the reaper is taking the anon anyway); an effective swappiness of 0 forces FILE-only regardless of the floor (the hard no-swap contract, which the `low_swappiness_mode` clamp never overrides since it only lowers — at the floor file is also blocked, so this OOMs rather than swaps, which is the intended behaviour); the floor-in-force condition forces ANON-only, outranking the proportional controller so a FILE-biased pick cannot get pinned on the floor-blocked side and stall reclaim at high effective swappiness. Otherwise the swap-bias proportional pick applies.

  The legacy orphan drain in `shrink_lruvec()` that follows the picker (draining folios that landed on `lruvec->lists` due to failed Marie installs or drain handoffs) mirrors the pick result via a `MARIE_DRAIN_*` bitmask returned by `lru_marie_shrink_lruvec()`. The drain zeroes `nr[]` for any type the picker did not scan, preventing `get_scan_count()`'s `SCAN_EQUAL` path (triggered at `sc->priority == 0`) from evicting file behind the picker's back during swap-fill at high effective swappiness.
- **No-progress OOM + swap-write-failure fast-path.**  Because the floor is enforced on every reclaim path, once clean file is at the floor and anon is unreclaimable reclaim returns no progress — so the kernel's stock `no_progress_loops` path in `should_reclaim_retry()` reaches the OOM killer promptly, with no separate free-memory watermark needed (this is exactly how le9uo converges at any floor size).  One Marie-specific addition remains in that slowpath: it also aborts the retry loop as soon as the swap backend has rejected more than `MAX_SWAP_WRITE_FAIL_RETRIES` writes during this allocation attempt — e.g. ZRAM `zs_malloc` starvation, where `can_reclaim_anon_pages()` still reports free slots that no longer accept writes, a case the floor alone would only resolve after grinding file down to it.
- **Honest reclaimable-anon accounting on RAM-backed swap.**  `should_reclaim_retry()`'s test is already exact and threshold-free: *if every reclaimable page were reclaimed, could any target zone reach its watermark?*  What was wrong on zram was its **input**. `zone_reclaimable_pages()` reported anon at face value, but on a RAM-backed swap device evicting a page frees it and immediately consumes a fraction of a page storing the compressed result, so the caller only nets `1 − 1/r` back. Sized far above RAM — as distros ship it — the device's nominal slots never run out, anon always *looks* reclaimable, the arithmetic always says a watermark is reachable, `no_progress_loops` keeps resetting, and the OOM killer never runs while every reclaim pass nets nothing. That is the RAM-filled-with-zspages freeze. Marie reports anon net of that cost:

  ```
  net = anon × (1 − zspages / stored)
  ```

  where `zspages` is `NR_ZSPAGES` and `stored` is the uncompressed page count currently held in swap (`total_swap_pages − get_nr_swap_pages()`). `r` is *measured*, not assumed — no ratio, fraction of RAM, or tuning constant appears anywhere — and it reaches zero exactly when the store has stopped compressing at all. Because it is the quotient of two large cumulative counters, a momentary allocation spike moves `anon` but barely moves the discount, so unlike a free-memory or headroom snapshot it cannot be tipped into a premature OOM by noise. Gated on `lru_marie_enabled()`, so MGLRU and Legacy builds keep vanilla arithmetic exactly. A side effect is that `MemAvailable` becomes honest on zram machines.
- **Thrash-livelock watchdog** (`mm/oom_kill.c`).  A 2-second delayed work item covering the case the arithmetic above cannot see: reclaim that *is* making nominal progress while everything it evicts faults straight back, the CPUs saturated in compression, free pinned between the min and high watermarks. With Marie enabled it accumulates a severity-weighted score per window — near-zero net progress and free in the reserves reach the threshold in a single window, a marginal treadmill needs the full 16 s — and any window where reclaim genuinely lifted free resets it. With Marie disabled it falls back to a PSI `mem-some` stall ratio (or, PSI off, a refault-vs-steal ratio), since this livelock reproduces on stock MGLRU too and is not a Marie property. On firing it invokes `out_of_memory()` directly and then backs off for 60 s.
- **`kcompressd` async swap-out.**  One `kcompressd<node>` kernel thread per node drains a `kfifo` of anon folios deposited by kswapd, running `zswap_store` / `__swap_writepage` off the reclaim critical path. The depth knob takes signed values from `-100` to `+100` (default `+24`). `0` disables the offload, positive values queue up to `|v|` folios and are gated by Marie's enabled state, and negative values force-enable the thread even when Marie is turned off.

### Coexistence

- **Static-key gate.**  Every Marie entry point is fronted by `lru_marie_enabled()`, a static key selected once at boot. Its compile-time default is `CONFIG_LRU_MARIE_DEFAULT_ON` (default `y`, giving a `STATIC_KEY_TRUE`); `N` builds it as `STATIC_KEY_FALSE` instead. Either way `lru_marie=0|1` on the kernel command line has the final say. When Marie is disabled the branch is patched out, so MGLRU / Legacy users see zero added instructions on the hot paths. There is no runtime on/off transition: the key is set before any folio is tracked, so there is no drain/fill machinery and no `transition_sem`-style serialisation on the reclaim path.
- **No folio->flags footprint.** Marie reserves no bits in `folio->flags`. All tracking state resides entirely inside the `marie_state` array, making its memory layout completely independent of core MM flags.
- **Marie off means off.** The changes Marie makes to shared mm code are gated on `lru_marie_enabled()` individually, not merely compiled in — including the `zone_reclaimable_pages()` discount and the watchdog's fast path. With `CONFIG_LRU_MARIE=n` every hook compiles out.

### Defragmentation (`CONFIG_LRU_MARIE_DEFRAG`, default y)

Marie's dense per-PFN array and aging generations happen to be exactly what a defragmenter wants, so an optional consumer reuses them: it scores 2 MiB pageblocks by the cost to evacuate them and clears only the few cold/unmapped stragglers of near-empty movable blocks — MOVE (reusing `migrate_pages()`) or, for cold-dead clean file, a cheap DROP.

When built in, the `defrag` switch (default `1`) makes Marie defrag **replace stock compaction inside kcompactd** on both paths — demand = MOVE + DROP, proactive = MOVE-only — and writing `0` falls back to stock kernel compaction at runtime. The cost when enabled is one atomic per Marie gen transition (install / aging / free) to maintain the per-pageblock occupancy histogram. With `CONFIG_LRU_MARIE_DEFRAG=n` all hooks compile out and the kernel image is identical to plain Marie.

Status: prototype. Verified safe — refault-free migration, bounded DROP, no corruption — but the yield-versus-stock A/B is real-hardware territory and has not been done. `defrag_drop` (default `1`) is a killswitch for just the DROP path; `defrag_stats` gates a cumulative diagnostic read.

---

## Tunables

All knobs live under `/sys/kernel/mm/lru_marie/`. Most accept runtime writes; `enabled`, `version` and `stats` are read-only (`enabled` reflects the boot-time selection — see `lru_marie=` below).

| Knob                          | Default       | Range / unit                | What it does                                                                                                 |
| :---------------------------- | :------------ | :-------------------------- | :----------------------------------------------------------------------------------------------------------- |
| `enabled`                     | `1`           | read-only (boot static key) | Master gate, selected at boot via `lru_marie=`. Read-only at runtime; `0` means Marie was patched out of the hot paths and MGLRU / Legacy took over. |
| `simd`                        | `1`           | 0 / 1, static key           | Toggles SIMD young-bit scan.  `0` falls back to scalar (A/B testing). Boot-settable via `lru_marie.simd=`.   |
| `simd_max`                    | `avx512`      | avx512 / avx2 / sse2        | Caps the SIMD ISA the walker may use, below what the CPU supports (e.g. avoid AVX-512 downclocking). Boot-settable via `lru_marie.simd_max=`. x86-only. |
| `version`                     | —             | read-only                   | Reports the current Marie LRU version.                                                                       |
| `stats`                       | —             | read-only                   | Exposes internal reclaim and walker statistics.                                                              |
| `clean_min_ratio`             | `10`          | 0–100, %RAM                 | Hard floor on clean file pages.  Below this, reclaim diverts to anon instead of evicting file.               |
| `low_swappiness_mode`         | `1`           | 0 / 1                       | When `1`, clamps the **effective** swappiness to at most 1, so distro / udev / tuning-daemon values above 1 are ignored by the pick driver. The clamp only lowers — a deliberate `vm.swappiness=0` is still honoured. Write `0` to obey the configured value verbatim and engage the proportional controller. |
| `kcompressd`                  | `24`          | -100..+100                  | Async swap queue depth. `0` disables. Positive = Marie-gated; negative = force-enable regardless of Marie. Present when `CONFIG_SWAP=y`. |
| `defrag`                      | `1`           | 0 / 1                       | Marie defrag replaces stock compaction inside kcompactd. `0` = fall back to stock compaction at runtime. `CONFIG_LRU_MARIE_DEFRAG` only. |
| `defrag_drop`                 | `1`           | 0 / 1                       | Killswitch for just the defrag DROP path (MOVE stays active). `CONFIG_LRU_MARIE_DEFRAG` only.                |
| `defrag_stats`                | `0`           | 0 / 1, then read            | Write `1` to enable the cumulative defrag diagnostic, then read the node for it. `CONFIG_LRU_MARIE_DEFRAG` only. |
| `walker_interval_critical_ms` | `HZ/30` (~33 ms) | ms                       | Walker cadence under critical memory pressure.                                                               |
| `walker_interval_low_ms`      | `HZ/10` (100 ms) | ms                        | Walker cadence under low pressure.                                                                           |
| `walker_interval_normal_ms`   | `HZ/4` (250 ms) | ms                         | Walker cadence under normal pressure.                                                                        |
| `walker_interval_idle_ms`     | `HZ` (1000 ms)  | ms                         | Walker cadence when idle.                                                                                    |

Generation size has no knob — it is derived from live occupancy per type; see [Gen-ring advance policy](#gen-ring-advance-policy-skip-on-full-no-forced-drain).

The `testing` ports (6.16 and later) carry one extra node not present on `stable`: `concede_pressure_mode` (0–3, default 3), an A/B selector for which pressure signal engages the FILE-then-ANON concede-to-anon — `1` = free level, `2` = file-refault feedback, `3` = either, `0` = floor only.

Boot cmdline: `lru_marie=0` disables Marie at boot (and `lru_marie=1` forces it on), overriding `CONFIG_LRU_MARIE_DEFAULT_ON` either direction. This is the only way to select Marie vs MGLRU / Legacy — the choice is latched into the static key before any folio is tracked, so the `enabled` node cannot be flipped at runtime. Two more boot-only parameters:

- `lru_marie_max_l2_pages_per_bit=N` — ceiling on PFNs per L2 bit (default 8192, ~32 MiB at 4 KiB pages). Consumed at `subsys_initcall` to size the bitmap and range-lock arrays, so it is deliberately not sysfs-tunable.
- `lru_marie.simd=0|1` and `lru_marie.simd_max=avx512|avx2|sse2` (x86) — seed the SIMD knobs before the first walker pass; overridden by any later sysfs write.

---

## Recommended Configuration

### The swappiness Paradigm: Storage Speed Matters

**On modern hardware (NVMe/SSD and ZRAM) there is nothing to set.** Marie clamps its own effective swappiness to at most 1 in-kernel via `low_swappiness_mode` (default `1`), so whatever value your distro, a udev rule, or a tuning daemon installed in `vm.swappiness` is simply ignored by the reclaim pick driver. Marie does **not** rewrite the kernel's `vm_swappiness` default — it stays at the upstream 60 — because the clamp makes that override unnecessary, and leaving the sysctl alone keeps its reported value honest for everything else that reads it.

The clamp only ever *lowers*: a deliberate `vm.swappiness=0` — the hard "never swap" contract — is still honoured verbatim.

**The Philosophy for Fast Storage:** Modern operating systems strive to keep physical RAM completely filled with file caches. When even slight memory pressure occurs, the upstream default (`swappiness=60`) dictates that the kernel should proportionally swap out anonymous pages. However, dropping a clean file page is instantaneous, whereas swapping out an anonymous page to ZRAM consumes significant CPU time, pollutes CPU caches with compression codecs, and blocks the calling context — causing visible UI stutter. With fast modern storage (NVMe/SSD), reading back a dropped file cache (refault) takes only microseconds and is virtually unnoticeable. Thus, forcing the system to grind on ZRAM swap-outs during normal daily usage just to preserve cold file caches is a severe waste of resources that degrades the user experience. Marie's `low_swappiness_mode` default encodes exactly this judgement.

**The Philosophy for Slow Storage:** If your system relies on **slower storage like HDDs**, the cost of reading back dropped file caches is high and causes severe unresponsiveness. Here you want the proportional controller instead — which takes **two** steps, because the clamp would otherwise flatten whatever you configure:

```
echo 0 > /sys/kernel/mm/lru_marie/low_swappiness_mode   # stop clamping
sysctl -w vm.swappiness=60                              # now honoured verbatim
```

With the clamp off, Marie honours values from 2 to 199 by engaging its proportional controller, balancing eviction between anon and file pages to minimize the severe I/O penalties of slow disks. Setting `vm.swappiness` alone, without clearing `low_swappiness_mode`, has no effect for any value above 1.

**The Synergy (at effective swappiness 1):** In Legacy LRU or MGLRU, running at `swappiness=1` is dangerous; it often leads to severe thrashing and livelocks because the kernel will evict the entire file cache (including active working sets) before it starts swapping. Marie LRU completely solves this via the `clean_min_ratio` hard floor (inherited from `le9uo`).

The default pairing of `low_swappiness_mode=1` with `clean_min_ratio=10` gives a robust synergy on modern systems:
1. **During normal use**, the kernel exclusively drops "cheap" file caches. Zero CPU time is wasted on premature anonymous swap-outs.
2. **Under severe pressure**, once the file cache drops to the `clean_min_ratio` floor, Marie securely locks the remaining file working set and forcibly diverts all reclaim pressure to anonymous pages (swap-out).

This guarantees that swap-out is strictly reserved for times of *genuine* memory exhaustion, delivering a vastly smoother desktop experience without the risk of thrashing.

*(Note for specialized servers: For environments that lock massive amounts of memory—such as large databases or virtualization hosts—it is recommended to lower `clean_min_ratio` to `5` or less. This prevents the file cache floor from overly restricting the remaining RAM available to your locked memory workloads.)*

### Disable `systemd-oomd`

It is highly recommended to **disable or mask `systemd-oomd`**. Marie LRU's internal memory management is robust under heavy pressure, with four independent mechanisms converging on "OOM promptly rather than freeze":

1. the `clean_min_ratio` floor (enforced on every reclaim path) lets reclaim reach a clean no-progress state at the floor, so the kernel's own OOM killer fires without thrashing;
2. `zone_reclaimable_pages()` reports anon net of the compressed store's RAM cost, so the kernel's no-progress test is not fed a fictional amount of reclaimable memory on zram;
3. a swap-write-failure fast-path deep inside the allocator slowpath (`should_reclaim_retry()`) catches ZRAM `zs_malloc` starvation instantly;
4. the thrash-livelock watchdog invokes the OOM killer on a sustained no-headroom treadmill, which is the one shape the arithmetic above cannot observe.

User-space OOM daemons like `systemd-oomd` often misinterpret Marie's proactive cache management as memory exhaustion and prematurely kill user applications.

⚠️ On very low-core or low-RAM machines a user-space PSI-based OOM daemon is still worth keeping — mechanisms 1–4 are kernel-side and cannot help if the CPUs themselves are saturated in compression before any of them arms.

---

## Build & install

To apply against a matching source tree:

```
cd /path/to/linux
patch -p1 < lru_marie.patch
make olddefconfig    # answer Y to CONFIG_LRU_MARIE
make -j$(nproc)
```

Three config symbols, all defaulting to `y`:

| Symbol | Default | Depends on | Effect |
| :--- | :--- | :--- | :--- |
| `CONFIG_LRU_MARIE` | `y` | `MMU` | Builds Marie. Memory cost ~1 byte per RAM PFN (4 MiB on 16 GiB, 16 MiB on 64 GiB). |
| `CONFIG_LRU_MARIE_DEFAULT_ON` | `y` | `LRU_MARIE` | Compile-time default of the boot static key. `N` = built in but off unless `lru_marie=1`. The command line always overrides either direction. |
| `CONFIG_LRU_MARIE_DEFRAG` | `y` | `LRU_MARIE && COMPACTION` | Marie defrag inside kcompactd. `N` compiles every hook out — the image is identical to plain Marie. |

`CONFIG_LRU_GEN` and `CONFIG_LRU_GEN_ENABLED` should stay as they were — Marie does not replace MGLRU at build time, only displaces it at boot (via the `lru_marie=` static key) when enabled.

To verify Marie is active after boot:

```
cat /sys/kernel/mm/lru_marie/enabled       # 1 = active
cat /sys/kernel/mm/lru_marie/version       # e.g., 0.10.0
```

---

## Status & roadmap

| Area                      | State            | Notes                                                                                                                |
| :------------------------ | :--------------- | :------------------------------------------------------------------------------------------------------------------- |
| x86-64 (AVX-512F)         | ✅ working        | preferred SIMD path                                                                                                  |
| x86-64 (AVX2 / SSE2)      | ✅ working        | auto-selected when AVX-512F absent                                                                                   |
| ARM64                     | ⚠️ scalar only    | NEON walker pending FPU save/restore profiling vs. the scalar baseline                                               |
| Other arches              | ⚠️ scalar only    | functional, no SIMD acceleration                                                                                     |
| Defrag (`CONFIG_LRU_MARIE_DEFRAG`) | ⚠️ prototype | verified safe (refault-free migration, bounded DROP, no corruption); yield-vs-stock A/B not yet run on real hardware |

Kernel bases carrying Marie 0.10.0 patches in this repo:

| Tier | Base |
| :--- | :--- |
| `patches/stable/` | 6.12.74 |
| `patches/testing/` | 6.16.12, 6.18.22, 7.1-rc5, 7.2-rc1 |

The `mm/lru_marie/` directory is version-agnostic: every in-tree mm API that changed signature across these kernels is isolated behind uniform names in three compat headers — `state_compat.h`, `walker_compat.h`, `defrag_compat.h` — switched by `LINUX_VERSION_CODE`, so producing a per-version port re-touches only those plus the unavoidable context of the integration hunks. The four `testing` ports are byte-identical outside `state_compat.h`; `stable` additionally lacks the `concede_pressure_mode` A/B knob, which exists from 6.16 onward. The API deltas are:

- `shrink_folio_list()` gained a trailing `memcg` parameter in **6.16** (not 6.18 — an earlier revision of this note said 6.18 and that mis-boundary broke the 6.13…6.17 range);
- `folio->flags` became the typed `memdesc_flags_t` (`struct { unsigned long f; }`) and `folio_pte_batch()` was replaced by `folio_pte_batch_flags()` in **6.18**;
- `arch_enter/leave_lazy_mmu_mode()` were renamed to `lazy_mmu_mode_enable/disable()` in **7.0** (compat provides the new names on 6.12 / 6.16 / 6.18);
- the batched no-flush young-PTE API `test_and_clear_young_ptes_notify()` is native on **7.1** (compat emulates it as a per-PTE `ptep_clear_young_notify()` loop on earlier bases);
- `PGSTEAL_*` / `PGSCAN_*` moved from `enum vm_event_item` to `enum node_stat_item` in **7.1**, so the same statistic must be read with `all_vm_events()` below that boundary and `global_node_page_state()` at or above it.

Because Marie is desktop/global-only, it carries no per-memcg lifecycle hooks, which removes the largest source of cross-version churn (notably the 7.1 objcg-reparent handling that earlier per-memcg revisions needed).

---

## Acknowledgements

Marie LRU builds upon the foundational ideas and pioneering work of several developers in the Linux memory management community. We express our deepest gratitude and respect to:

- **Yu Zhao (Google)**: The original author of **MGLRU (Multi-Gen LRU)**. MGLRU revolutionized Linux page reclaim by introducing the generational ring, the active PTE walker, and the PMD Bloom filter. Marie LRU's foundational concepts — specifically generational aging, background page table scanning, and Bloom filter-driven hot-signal harvesting — are profoundly influenced by MGLRU's trailblazing design.
- **Alexey Avramov (a.k.a. hakavlad)**: The original author of [le9](https://github.com/hakavlad/le9-patch). Marie's `clean_min_ratio` working-set protection and pressure-resilience philosophies are the direct spiritual successors to the le9 project.
- **Qun-Wei Lin (MediaTek)**: The original author of the **Kcompressd** concept. Marie's `kcompressd` asynchronous swap-out offload is heavily inspired by their architectural design to decouple reclaim from `zswap_store`.

### Predecessor patches

[firelzrd/le9uo](https://github.com/firelzrd/le9uo)
[firelzrd/Re-swappiness](https://github.com/firelzrd/re-swappiness)
[firelzrd/kcompressd-unofficial](https://github.com/firelzrd/kcompressd-unofficial)

## License

GPL-2.0, same as the Linux kernel.
