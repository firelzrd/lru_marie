# Marie LRU

**Multi-graded Adaptive Reclaim & Independent Eviction**

> [!NOTE]
> **Marie's successor, [Cost-Aware Self-Tuning LRFU (lru_cast)](https://github.com/firelzrd/lru_cast), is now available** for Linux 6.12, 6.16, 6.18, 7.2 and 7.3. It replaces Marie's generational ring with a self-tuning LRFU that measures its own decay, newcomer window and anon/file split — swappiness included — and evicts more accurately: in our tests it reads back less than half as many pages from swap as MGLRU.
>
> **Test reports are wanted.** If you try it, please share how it went — workload, hardware, and what you saw — in the [lru_cast issues](https://github.com/firelzrd/lru_cast/issues).

Marie is an out-of-tree, third-pillar LRU implementation for the Linux kernel, sitting alongside Legacy LRU and MGLRU rather than replacing either.  It composes three established ideas — working-set protection ([le9uo](https://github.com/firelzrd/le9uo)), proportional anon/file aging under a generational ring ([Re-swappiness](https://github.com/firelzrd/re-swappiness)), and async swap-out offload ([kcompressd-unofficial](https://github.com/firelzrd/kcompressd-unofficial)) — onto a redesigned core: a flat per-PFN state array (one 32-bit word per PFN) with no linked lists or staging queues, a single global generation ring per memory type without forced oldest-gen drain, and a SIMD-accelerated PTE walker driven by rmap-to-walker bloom-filter feedback. Two mechanisms then fall out of that core rather than being bolted onto it: reclaim accounting derived from state-word transitions, and an optional defragmenter that reuses the same array and generations in place of stock compaction.

Marie is **desktop / global-only** in its aging: it runs a single node-global aging clock and reclaim scan, one generation ring per memory type shared by every cgroup, and keeps *no* per-memcg or per-lruvec reclaim state (no per-cgroup lists, generations or bitmaps). cgroup *charging* (`memory.current`, PSI) stays on the stock path. A cgroup that reaches its own limit is still reclaimed: every tracked folio carries the slot of the memcg it is charged to in its state word, and that cgroup's reclaim takes only its subtree's folios out of the shared ring (see [cgroup-targeted reclaim](#cgroup-targeted-reclaim)). What this does not give is per-cgroup aging or protection — `memory.low` / `memory.min` are not honoured by Marie's global scan — so it is unsuited to containers, servers or Android setups that depend on per-cgroup reclaim isolation. For a single-user desktop, Marie's near-OOM stability covers the purpose of cgroup limits and `oom_score_adj` covers OOM targeting.

Marie is selected once at boot behind a static key (`lru_marie_enabled()`), so a kernel built with `CONFIG_LRU_MARIE=y` but booted with `lru_marie=0` behaves exactly as MGLRU / Legacy LRU would. There is no runtime on/off toggle — the key is set before any folio is tracked, so the `enabled` sysfs node is read-only.

---

## Background

Linux's page-reclaim LRU has gone through two main eras.

**Legacy LRU** (the upstream default before MGLRU, and still the fallback) maintains per-memcg active/inactive list pairs.  Folios are promoted on `PG_referenced` and demoted on scan-tail age-out.  The model is small and cheap, but the binary active/inactive boundary is coarse — within each bucket there is no further ordering, so when the inactive list isn't large enough to span the working set, refault-driven re-promotion turns into a thrash treadmill.

**MGLRU** (`lru_gen`, mainlined in 6.1) replaced the two-bucket model with a generational ring, actively harvested by a PTE walker rather than relying solely on rmap-side `PG_referenced`.  This delivered a large improvement on workloads where the active/inactive cut sliced through the working set, and remains the right baseline for most general-purpose use.

Marie is a third design point.  The axes where MGLRU's choices turn into trade-offs under specific workloads — and where Marie picks the other end:

Legend — ✅ behaves as expected · ⚠️ works but with a structural trade-off · ❌ not provided / cannot deliver the property

| Aspect                | Legacy LRU                                                                | MGLRU                                                                                                       | Marie |
| :-------------------- | :------------------------------------------------------------------------ | :---------------------------------------------------------------------------------------------------------- | :---------------------------------------------------------------------------------------------------------- |
| Aging granularity     | ⚠️ binary active / inactive only                                           | ✅ 4-gen ring (`MAX_NR_GENS=4`); a full ring triggers a forced `inc_min_seq` oldest-gen drain                 | ✅ 8-gen ring (`MARIE_PFN_NR_GENS=8`); advance skips when the ring is full, no forced drain |
| anon/file iteration   | ✅ per-type lists; `vm.swappiness` works as a proportional anon:file ratio | ❌ shared `max_seq` across anon/file reduces `vm.swappiness` to a per-generation hint                        | ✅ per-type `seq` / `min_seq` / ring; restores `vm.swappiness` as a proportional ratio |
| Hot-signal harvest    | ⚠️ rmap-driven `PG_referenced` only; no active walker                      | ✅ PTE walker + walker-internal PMD bloom filter + rmap look-around cascading `PG_referenced`                | ✅ Vectorized PTE walker + rmap-fed PMD bloom filter + rmap look-around that promotes straight to the head gen and cascades no `PG_referenced` |
| Compression stall     | ❌ kswapd blocks inside `zswap_store`                                      | ❌ kswapd blocks inside `zswap_store`                                                                        | ✅ offloaded to per-node `kcompressd` kthread |
| Clean-file floor      | ❌ none                                                                    | ❌ none                                                                                                      | ✅ `clean_min_ratio` hard floor against thrashing |
| Reclaim-livelock OOM  | ❌ retries until `MAX_RECLAIM_RETRIES`                                     | ❌ retries until `MAX_RECLAIM_RETRIES`                                                                       | ✅ four conditions (swap store starved, a turnover that freed no RAM, swap heat over `oom_heat_limit`, plus the stock allocator no-progress path made reachable by the `clean_min_ratio` floor) — see [When Marie invokes the OOM killer](#when-marie-invokes-the-oom-killer) |
| RAM-backed swap (zram) | ❌ anon always *looks* reclaimable → OOM may never arm                     | ❌ same                                                                                                      | ✅ a store that cannot take a page, or that takes as much RAM as eviction frees, is detected from the swap write completion and the live `zspages` counter — no ratio, no tuning constant |
| Defragmentation       | ⚠️ stock compaction                                                        | ⚠️ stock compaction                                                                                          | ✅ optional (`CONFIG_LRU_MARIE_DEFRAG`) — reuses the per-PFN array + gens to score and clear near-empty pageblocks inside kcompactd |
| cgroup reclaim / isolation | ✅ per-memcg active/inactive lists; `memory.max` enforced via targeted reclaim | ✅ per-memcg gen state; `memory.max` enforced via targeted reclaim | ⚠️ one global ring; a cgroup at its limit is reclaimed through owner tags (only its own folios are taken, the node-wide `clean_min_ratio` floor does not apply to it). No per-cgroup aging, no `memory.low` / `memory.min` |
| Memory hotplug        | ✅                                                                         | ✅                                                                                                           | ✅ per-section attach/detach of the state array and scan planes; DIMM, virtio-mem and balloon tested in QEMU |

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
             │ marie_state[pfn]           │  flat 32-bit per-PFN array
             │  (TRACKED, ISOLATED,       │  (no linked lists, no staging)
             │   TYPE, ZONE, GEN,         │
             │   REFS, EPOCH, OWNER)      │
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

- **Flat per-PFN state array (`marie_state`)**: Marie abandons linked lists entirely. Every folio's state is one 32-bit word (TRACKED, ISOLATED, TYPE, ZONE, GEN in the low byte; REFS, EPOCH and the owner memcg slot above it) in a flat array whose address space is reserved once at boot and backed per memory section. This eliminates allocation in the fast paths and per-CPU staging queues.
- **L1/L2 tracking bitmaps**: To efficiently scan the flat array, Marie maintains one summarized L2 bitmap per `(type, zone, gen)` plane — 64 planes in all, each sized to its zone's PFN span — so the isolate scanner skips whole empty PFN ranges in a single cycle. The granularity is chosen at boot: a plane targets ~512 bits (`MARIE_L2_TARGET_BITS`, "a handful of cache lines"), capped so one L2 bit never covers more than 8192 pages (~32 MiB at 4 KiB), which is what bounds the per-hit L1 skim on large machines. Boot-tunable via `lru_marie_max_l2_pages_per_bit=`.
- **Independent per-type rings**: Aging is tracked by `marie_head_gen[type]` cycling counters. Anon and file have fully independent rings, restoring swappiness as a proportional scan ratio without forcing oldest-gen drains.
- **ISOLATED as an ownership token**: bit 6 marks a PFN as exclusively owned by an in-flight isolate/putback. Every other GEN mutator — walker promote-on-access, `MADV_COLD`, `MADV_FREE`, defrag restamp — must check it first and decline. This is also what makes the reclaim accounting exactly-once (see below).
- Hot-signal pickup is split across rmap and the walker. MGLRU also uses a bloom filter, but it's walker-internal — a self-feedback to skip PMDs that the walker itself did not touch on the previous pass. Marie's bloom is cross-component: rmap is the producer (the PTL-bounded look-around flags PMDs that just took a young hit), the walker is the consumer (it pays the SIMD scan + promotion cost only on bloom-hit PMDs). This lets a hot signal observed by rmap influence the walker within the same reclaim window, not the next one.

---

## Design

### Reclaim pipeline: Flat per-PFN state array

Marie tracks a folio's lifecycle entirely via one 32-bit word per PFN in `marie_state[pfn]`. The low byte is the lifecycle and placement state, ordered MSB → LSB by hierarchy rather than convenience:

| Bits | Field | Meaning |
| :--- | :--- | :--- |
| 7 | **TRACKED** | 1 = owned by Marie; 0 = the low byte means nothing. The single source of truth — no `folio->flags` bit is used. |
| 6 | **ISOLATED** | 1 = exclusively owned by an in-flight isolate/putback. Every other GEN mutator checks this and declines. |
| 5 | **TYPE** | 1 = file LRU, 0 = anon LRU |
| 4..3 | **ZONE** | `folio_zonenum` (DMA / DMA32 / NORMAL / MOVABLE) |
| 2..0 | **GEN** | position 0..7 in the cycling ring (relative to `marie_head_gen[type]`) |

The upper 24 bits carry what the placement rule and cgroup reclaim need:

| Bits | Field | Meaning |
| :--- | :--- | :--- |
| 11..8 | **REFS** | in how many distinct epochs the folio has been seen accessed (saturating at 7 by default) — its reuse class (see [Placement](#placement-reuse-classes-and-an-adaptive-position)) |
| 14..12 | **EPOCH** | the epoch of the last counted access (3 bits, compared for equality) |
| 15 | **RESTORED** | on an untracked PFN: a note that the next install takes over the REFS the folio carried in its shadow entry |
| 31..16 | **OWNER** | slot of the memcg the folio is charged to (0 = none), so a cgroup's reclaim can pick its folios without chasing `folio->memcg_data` |

TRACKED and ISOLATED are gates — existence, then ownership — that must be read before anything else in the word is meaningful. Untracked PFNs read as 0 (or as a RESTORED note).

There is no hotness counter in the GEN field: a folio whose reuse is counted is promoted straight to the head gen by a single CAS on the word. (`marie_state_inc_tier()` keeps its historical name.)

Because state is purely array-based, there are no `list_del` or `swap-pop` operations during eviction or installation. Zeroing the word instantly drops tracking. To isolate folios, Marie uses an **L2-pruned `__ffs`/`blsr` walk** combined with **Two-Stage Software Prefetching**:
- **L2-pruning**: Scanning the `(type, zone, gen)` L2 bitmap for the target bucket skips empty PFN ranges (up to ~32 MiB per bit) in 1 CPU cycle.
- **Hardware Extraction**: Surviving L2 blocks are resolved to precise PFNs in near O(1) time using `__ffs` (Find First Set) and `blsr` (reset lowest set bit) hardware instructions over the L1 bitmap.
- **Strategic Prefetch**: A "Cache-line cursor" tracks boundaries to issue exactly one prefetch per cache-line. It fires `prefetcht2` (DRAM → L3) far ahead of the processing cursor, and `prefetcht0` (L3 → L1) slightly ahead, hiding the DRAM latency wall without thrashing the L1 cache.

### One state machine, one writer

What can happen to a PFN's word is a closed set of ten events — NOTE, INSTALL, INSTALL_ISOLATED, ACCESS, MOVE, FOLD, ISOLATE, PUTBACK, RETAG, UNTRACK — and what the word becomes is decided by a pure function of the old word and the event (`model/fsm`). An event that does not fit the word's state (a MOVE of an isolated folio, an ISOLATE of an untracked one) is declined and changes nothing. `marie_apply()` is the only place the array is written: it applies the decision atomically and keeps the occupancy, the folio count, the scan index and the owner filters in step with the transition that actually happened. The pure model is checked exhaustively in userspace (every old word of the low 16 bits against every event, and random walks against a recount), without a kernel.

### Accounting: derived from state transitions, not from call sites

Marie's LRU size counters (`marie_nr_folios`, the per-`(gen, type)` occupancy, `__update_lru_size`) are never adjusted by whichever code path happens to be running. Each is derived from the *committed* transition of the state word (`marie_fsm_occ()` in the pure model computes the difference from the old and new word):

```
d = pred(new_byte) - pred(old_byte)      where pred(b) = TRACKED(b) && !ISOLATED(b)
```

`pred()` means "this PFN currently holds an outstanding +1". A path that publishes a state it has already published contributes `d = 0`; a path that debits a folio some other path already debited contributes `d = 0`. Double-counting is therefore not a bug to be detected but an unrepresentable state, and ISOLATED doubles as the exactly-once token for the uncharge backstop that runs at `folio_put()` time. This replaced a family of hand-written per-site adjustments in which one miscounting branch was enough to drive the LRU counters negative and cause premature OOM.

### Gen-ring advance policy: skip on full, no forced drain

MGLRU's ring has `MAX_NR_GENS=4`.  When it fills, aging calls `try_to_inc_min_seq()` to push the oldest gen forward before cutting a new head.  If that oldest gen holds folios that won't move — clean file pages under a working-set policy, mlocked anon, etc. — aging still demands forward progress, so the same untouchable folios get rewalked over and over: the *forced `inc_min_seq` treadmill*.

Marie uses an 8-gen cap (`MARIE_PFN_NR_GENS`, from 3 GEN bits) tracked by `marie_head_gen[type]`. Head-generation advance is *drain-wait gated*: `marie_try_advance_head_mlv()` will not advance if the next slot in the ring still contains folios of that type.

Practical effect: an oldest gen full of `clean_min_ratio`-protected folios is left alone.  Aging stops growing new heads until reclaim drains the existing tail (or `clean_min_ratio` diverts the type selector to anon).  The treadmill never starts because nothing in Marie demands aging-side progress regardless of consumer state.

**Why 8 gens.** Spending the third GEN bit on the ring rather than on a hotness field lengthens a folio's head→oldest descent — its time-domain reclaim grace — giving the walker's hot-signal harvest more passes to re-promote a *warm* folio, one re-accessed on a period comparable to that descent, before it reaches reclaim range. In a warm-set A/B this cut warm-folio refaults ~40% versus a 4-gen ring with a 4-level hotness field. A 4-gen / 2-level control did not reproduce the gain, which isolates it to the gen count rather than to the promotion threshold — so the hotness field was retired outright and promote-on-access now goes straight to the head gen.

**Gen size is automatic.** A new head is cut once the current one has taken `thr` pages' worth of installs. That threshold is computed, not configured — the only knob is `gen_floor_shift`, a debugging aid left at 0:

```
thr = max( base >> (NR_GENS + 1 - gens),  base >> (log2(NR_GENS) + gen_floor_shift) )
```

`base` is what the type could grow into (its own pages plus free memory), refreshed at each head advance and at reclaim entry; `gens` is how many of the type's generations hold anything right now. Few occupied generations shift far and advance quickly (the ring fills); many shift little and advance slowly (reclaim gets time to drain the oldest). The floor stops the cadence from running faster than one generation per 1/8 of `base`. The equilibrium sits at 4.7–6.3 occupied generations, always short of 8, so a spare slot survives and the head keeps moving. Installs are counted in pages, not folios, so THP does not skew the cadence.

**A gen that cgroup reclaim leaves behind is folded.** The head can only advance into an *empty* slot. A global reclaim drains the oldest gen, but a cgroup's reclaim takes only its own folios, so whatever other cgroups left in the oldest gen would stay there and stop the ring (measured: with one cgroup at its limit the head sat still and all of the cgroup's pages piled into one gen). When a cgroup's reclaim finds that nothing of its own is in the oldest gen, that gen is folded into the next one (the FOLD event: a resident folio moves only if it is still in the gen it is folded out of). No folio is evicted or reordered; the folded folios stay older than everything installed since.

### Independent anon/file pipelines

Marie keeps **fully independent** per-type state — each of ANON and FILE has its own `marie_head_gen[type]` cycling counter, install cadence and tracking bitmaps. There is no shared iteration counter between the two types, so swappiness works as a proportional anon:file scan ratio — the behaviour Legacy LRU has always had, recovered inside a generational ring. (Note that with the default `max_swappiness=1` the *effective* swappiness is capped at 1, which selects the FILE-then-ANON regime rather than the proportional controller; see [Recommended Configuration](#recommended-configuration).)

### Placement: reuse classes and an adaptive position

Reclaim always takes the oldest non-empty generation and skips nothing. How long a folio lives is therefore decided where it is *put*, once, by one rule used for install, access, putback and refault.

- **Two classes.** A folio whose reuse has been seen (REFS ≥ 1) is class F; one not yet reused (REFS = 0) is class U. A reuse is the first access in a later *epoch* — an epoch being 1/8 of the type's resident pages, counted in installed pages (so it does not stall when the ring does) — or a young mapping found on a reclaim survivor, or a refault inside the ghost window.
- **Placement.** F goes to the head gen. U goes to position `p` of the eviction queue: the first generation that is younger than the oldest non-empty one and has at least `p` pages in the generations older than it. A U folio is never put into the oldest non-empty generation — that one is only drained, so it always empties and the head can advance; fed as well, it never would and the ring would stop turning.
- **`p` is not tuned; it follows what evictions cost**, the way ARC's target does. A folio that comes back inside the ghost window (`distance ≤` the pages the type holds) was evicted too early: if it was evicted as U, U waited too short a time and `p` grows; if as F, F's residence was the waste and `p` shrinks. The step is the ratio of the other class's eviction rate to this one's (at least 1), `p` stays within `[0, c]`. A refault inside the window returns with its saved REFS + 1, i.e. as F, at the head. REFS rides in the shadow entry (the top bits of the eviction counter), so the class survives eviction.
- Class counts and `p` are visible in `/sys/kernel/mm/lru_marie/place`; the access statistics in `refs_stats` / `refs_dist` (enable with `refs_track`).

Measured in QEMU with a Zipf workload under global pressure, refaults per 100 accesses fell about 5% against 0.12.0 (and about 10% in a 1.5 GiB cgroup). Hardware numbers have not been taken.

### cgroup-targeted reclaim

Aging stays global; a cgroup that reaches its own limit is reclaimed by selection. Each tracked folio's state word carries the slot of the memcg it is charged to (up to 65535 slots; `lru_marie_owner_max=` lowers the cap for testing — folios of memcgs beyond it are untagged and still reclaimable through the authoritative path). A cgroup's reclaim builds a filter of its subtree's slots and uses two indexes to keep the scan cheap: a presence filter per `(type, gen)` that says whether any of its owners may be in that generation, and a per-64-PFN block index. The tag is only a prefilter — `folio_memcg()` stays authoritative. The node-wide `clean_min_ratio` floor is a reserve against *global* pressure and does not apply to a cgroup at its own limit, or `memory.max` could not be enforced. `owner_audit` (write 1, read) cross-checks the tags against the folios on a quiescent system; `owner_verify` makes the scan look at what the index hid and count it.

### Memory hotplug

`marie_state`, the owner indexes and the scan planes are sized for the largest PFN the machine can have (`max_possible_pfn`, capped at 2^32) in *address space*, and backed by pages one 128 MiB memory section at a time. A memory notifier attaches a section's state when it goes online and detaches it on offline, after taking the section's L2 range locks and clearing its counts and bits so a stale scan bit can never reach unmapped memory. If a section cannot be attached, its folios stay on the legacy list. Tested in QEMU with DIMM add/remove, two DIMMs, virtio-balloon and virtio-mem.

### Hot-signal harvest

- **SIMD PTE walker** (per-pgdat).  On x86-64, `arch_initcall` picks the widest available SIMD instruction set (AVX-512F > AVX2 > SSE2) — capped by the `simd_max` knob — and flips a static branch so the walker's young-bit extraction has no indirect call. Each scan reads 512 64-bit PTEs at once and reduces them to a 512-bit young-bit mask via vector ops, entirely avoiding per-PTE branching. Both `simd` (on/off) and `simd_max` (ISA cap) are runtime-tunable and boot-seedable. ARM64 and other arches use a scalar fallback for now.
- **Per-PMD FPU bracket, promotion drained outside the locks**.  `kernel_fpu_begin/end` wraps only the SIMD scan of a single PMD and never spans PMDs or bloom-miss stretches, so there is no batching knob to tune: save/restore is skipped entirely for kthreads (kswapd) and costs at most one save per kernel entry otherwise, which makes per-PMD bracketing effectively free. The heavy per-young-PTE work is then split in two — the `ptep_test_and_clear_young` aging runs under the PTL because it must, collecting young PFNs into a per-CPU worklist, and the head-gen promotions are drained *after* the PTL is dropped. The dominant cost therefore runs lock-free, preempt-enabled, and outside the FPU bracket.
- **rmap look-around**.  Called from `folio_referenced_one()`, this scans up to `BITS_PER_LONG` PTEs around the target folio's PMD under the existing PTL.  Unlike MGLRU's look-around it does **not** set `PG_referenced` on neighbouring folios — that cascade is the main source of reclaim starvation under fault-heavy workloads.  Instead it feeds the walker via a bloom filter (below).
- **rmap-fed PMD bloom feedback**.  MGLRU's bloom filter is walker-internal: the walker remembers which PMDs it touched last pass and uses that to skip cold PMDs next pass.  Marie's bloom is cross-component — rmap is the producer ("this PMD had a young PTE on its target"), the walker is the consumer ("only scan PMDs the bloom flagged").  The filter is double-buffered and swapped at the end of each walker pass, so each pass acts on the rmap signal accumulated during the previous reclaim window.
- **Pressure-adaptive cadence**.  The walker re-evaluates its scan interval (`walker_interval_{critical,low,normal,idle}_ms`) on each pass.  Defaults: `HZ/30`, `HZ/10`, `HZ/4`, `HZ` — all tunable.

### Pressure resilience

- **`clean_min_ratio` (hard working-set floor).**  Marie withholds file reclaim — on *every* reclaim path (the per-PFN pick driver and the legacy drain that follows it in `shrink_lruvec`) — when clean file pages would otherwise be evicted below the configured percentage of node RAM, diverting reclaim to anon instead. Only clean file counts toward the floor: dirty pages, which cannot be reclaimed without writeback, are subtracted so they cannot satisfy the floor on unreclaimable pages. Equivalent in spirit to le9uo's knob of the same name.  Default 10%; set to 0 to disable. Two reclaimers are outside it by design: an OOM victim (a task certain to give its memory back may use the file below the floor to finish exiting) and a cgroup reclaiming for its own limit (see [cgroup-targeted reclaim](#cgroup-targeted-reclaim)). Nothing else is exempt — in particular there is no rescue that draws on the floor for a task that lives.

  The pick driver selects the reclaim type via a strict priority cascade: OOM victims and anon-unreclaimable conditions force FILE-only (no point swapping if no slots exist or the reaper is taking the anon anyway); an effective swappiness of 0 forces FILE-only regardless of the floor (the hard no-swap contract, which the `max_swappiness` cap never overrides since it only lowers — at the floor file is also blocked, so this OOMs rather than swaps, which is the intended behaviour); the floor-in-force condition forces ANON-only, outranking the proportional controller so a FILE-biased pick cannot get pinned on the floor-blocked side and stall reclaim at high effective swappiness. Otherwise the swap-bias proportional pick applies.

  The legacy orphan drain in `shrink_lruvec()` that follows the picker (draining folios that landed on `lruvec->lists` due to failed Marie installs or drain handoffs) mirrors the pick result via a `MARIE_DRAIN_*` bitmask returned by `lru_marie_shrink_lruvec()`. The drain zeroes `nr[]` for any type the picker did not scan, preventing `get_scan_count()`'s `SCAN_EQUAL` path (triggered at `sc->priority == 0`) from evicting file behind the picker's back during swap-fill at high effective swappiness.
- **OOM that converges.**  Because the floor is enforced on every reclaim path, once clean file is at the floor and anon is unreclaimable reclaim returns no progress, and the kernel's stock `no_progress_loops` path in `should_reclaim_retry()` reaches the OOM killer promptly (this is how le9uo converges at any floor size). The cases that path cannot see — a swap store with no page left to take a write, evictions that free no RAM, a swap treadmill that holds up the tasks that want to run — are judged by a watchdog that runs only while there is swap load. All conditions are listed in [When Marie invokes the OOM killer](#when-marie-invokes-the-oom-killer).
- **`kcompressd` async swap-out.**  One `kcompressd<node>` kernel thread per node drains a `kfifo` of anon folios deposited by kswapd, running `zswap_store` / `__swap_writepage` off the reclaim critical path. The depth knob takes signed values from `-100` to `+100` (default `+24`). `0` disables the offload, positive values queue up to `|v|` folios and are gated by Marie's enabled state, and negative values force-enable the thread even when Marie is turned off.

### When Marie invokes the OOM killer

Reclaim goes on while three predicates hold; the node is out of memory when any one fails, or when the swap meter is too hot:

> OOM ⇔ ¬Capable ∨ ¬Feasible ∨ ¬Productive ∨ ¬Cool

The first three ask whether the node *can* recover and have no threshold in them. The last asks how long it may go on being slow; it is a policy, and its one number is a duration in seconds that the operator sets.

| Condition | Fires when | Judged by |
| :--- | :--- | :--- |
| **Capable** | a RAM-backed swap device (zram…) found no page to put a write in, even from inside reclaim where it may use the reserves. Fires at once, without waiting for the next interval. | the swap write completion (`mm/page_io.c`) → the watchdog |
| **Feasible** | reclaiming everything reclaimable could not bring any target zone to its watermark — anon counted at face value, clean file withheld by the floor not counted. | stock `should_reclaim_retry()` (allocator slowpath) |
| **Productive** | a *turnover* of evictable memory (all anon plus the file above the floor) has been evicted and the pages evicted did not exceed the RAM the compressed store took for them (`evicted − zspages grown ≤ 0`): evicting frees nothing. | the watchdog, once per turnover |
| **Cool** | the *swap heat* reached `oom_heat_limit` seconds without a break (below). | the watchdog, every interval |

**Swap heat.** Swapping is productive when what goes out stays out. Per interval:

- `ρ` = the share of the pages swapped out that came back because a task faulted on them (`PSWPIN − SWAP_RA` over `PSWPOUT`; pages a readahead fetched on spec are not counted); 0 when nothing went out. `ρ = 0` is swap doing its job, `ρ = 1` is a treadmill.
- `h` = the share of the tasks that want the CPU that are stalled on memory, `S / (S + R)`, from PSI's per-CPU counters (tasks sleeping on anything else are not counted). 1 for the only task there is, 1/64 for one of sixty-four, 0.04 for forty of a thousand — independent of the number of CPUs.
- `heat' = max(0, heat + dt · (ρ·h − (1 − ρ)))`, a rate between −1 and +1. Swap that keeps freeing memory cools however large it is; a treadmill that holds up the tasks that want to run heats at the rate it holds them up. The break-even `ρ = 1/(1 + h)` is where as many pages come back as stay out — the same unit on both sides, not a chosen ratio.

The meter exists only while there is swap load: the first anon reclaim starts the watchdog timer (2 s), and when the load has ended — nothing evicted, nobody stalled — the meter is reset and the timer stops. So the heat is only ever the heat of thrashing that has gone on without a break, and an idle machine runs no timer at all. `oom_heat_limit` (default 30, `0` turns the meter off and leaves the other three) is the only knob; `oom_stats` shows the ticks judged, the current heat and `peak_ms`, the highest heat since boot — read it under your real load when choosing the limit. Without PSI the meter never heats. For reference, in QEMU (4 GiB, zram): `global`-style overload that completes and a 3.5 GiB hog stay at 0; a 12-worker thrash at PSI-some ~60% peaks near 9 s; a single task touching 3.8 GiB at random (PSI-full ~65%) heats at about 0.27 s/s, i.e. 30 s is reached after about two minutes of it. These are QEMU figures, not hardware.

**Not triggers.** A node that is slow but productive; a swap burst that ends; many tasks running while a few are stalled; a victim that is already exiting (`out_of_memory()` declines a new kill). **Who is killed** is the stock `oom_badness()` choice, not necessarily the task that is stalled.

### Coexistence

- **Static-key gate.**  Every Marie entry point is fronted by `lru_marie_enabled()`, a static key selected once at boot. Its compile-time default is `CONFIG_LRU_MARIE_DEFAULT_ON` (default `y`, giving a `STATIC_KEY_TRUE`); `N` builds it as `STATIC_KEY_FALSE` instead. Either way `lru_marie=0|1` on the kernel command line has the final say. When Marie is disabled the branch is patched out, so MGLRU / Legacy users see zero added instructions on the hot paths. There is no runtime on/off transition: the key is set before any folio is tracked, so there is no drain/fill machinery and no `transition_sem`-style serialisation on the reclaim path.
- **No folio->flags footprint.** Marie reserves no bits in `folio->flags`. All tracking state resides entirely inside the `marie_state` array, making its memory layout completely independent of core MM flags.
- **Marie off means off.** The changes Marie makes to shared mm code are gated on `lru_marie_enabled()` individually, not merely compiled in — including the floor-aware `zone_reclaimable_pages()` and the OOM watchdog. With `CONFIG_LRU_MARIE=n` every hook compiles out.

### Defragmentation (`CONFIG_LRU_MARIE_DEFRAG`, default y)

Marie's dense per-PFN array and aging generations happen to be exactly what a defragmenter wants, so an optional consumer reuses them: it scores 2 MiB pageblocks by the cost to evacuate them and clears only the few cold/unmapped stragglers of near-empty movable blocks — MOVE (reusing `migrate_pages()`) or, for cold-dead clean file, a cheap DROP.

When built in, the `defrag` switch (default `1`) makes Marie defrag **replace stock compaction inside kcompactd** on both paths — demand = MOVE + DROP, proactive = MOVE-only — and writing `0` falls back to stock kernel compaction at runtime. The cost when enabled is one atomic per Marie gen transition (install / aging / free) to maintain the per-pageblock occupancy histogram. With `CONFIG_LRU_MARIE_DEFRAG=n` all hooks compile out and the kernel image is identical to plain Marie.

Status: prototype. Verified safe — refault-free migration, bounded DROP, no corruption — but the yield-versus-stock A/B is real-hardware territory and has not been done. `defrag_drop` (default `1`) is a killswitch for just the DROP path; `defrag_stats` gates a cumulative diagnostic read.

---

## Tunables

All knobs live under `/sys/kernel/mm/lru_marie/`. Most accept runtime writes; `enabled`, `version` and `stats` are read-only (`enabled` reflects the boot-time selection — see `lru_marie=` below).

| Knob                          | Default       | Range / unit                | What it does |
| :---------------------------- | :------------ | :-------------------------- | :----------------------------------------------------------------------------------------------------------- |
| `enabled`                     | `1`           | read-only (boot static key) | Master gate, selected at boot via `lru_marie=`. Read-only at runtime; `0` means Marie was patched out of the hot paths and MGLRU / Legacy took over. |
| `simd`                        | `1`           | 0 / 1, static key           | Toggles SIMD young-bit scan.  `0` falls back to scalar (A/B testing). Boot-settable via `lru_marie.simd=`. |
| `simd_max`                    | `avx512`      | avx512 / avx2 / sse2        | Caps the SIMD ISA the walker may use, below what the CPU supports (e.g. avoid AVX-512 downclocking). Boot-settable via `lru_marie.simd_max=`. x86-only. |
| `version`                     | —             | read-only                   | Reports the current Marie LRU version. |
| `stats`                       | —             | read-only                   | Exposes internal reclaim and walker statistics. |
| `clean_min_ratio`             | `10`          | 0–100, %RAM                 | Hard floor on clean file pages.  Below this, reclaim diverts to anon instead of evicting file. |
| `max_swappiness`              | `1`           | 0–200                       | A ceiling on the **effective** swappiness: the pick driver uses `min(vm.swappiness or memory.swappiness, max_swappiness)`, so distro / udev / tuning-daemon values above the cap are ignored. `1` is strict file-first (file down to the `clean_min_ratio` floor, then anon). The cap only lowers — a deliberate `vm.swappiness=0` is still honoured. Write `200` to remove the cap and obey the configured value verbatim (engaging the proportional controller for 2–199). |
| `oom_heat_limit`              | `30`          | seconds, 0 = off            | How long the swap meter may stay hot, in seconds of swap that bought nothing, before the node is judged out of memory. See [When Marie invokes the OOM killer](#when-marie-invokes-the-oom-killer). |
| `oom_stats`                   | —             | read-only                   | `ticks` judged (0 while there is no swap load), current `heat_ms`, `peak_ms` (highest since boot), the limit, and whether an episode is active. |
| `place`                       | —             | read-only                   | The placement rule's state per type: `p`, the position of class U, its generation, the class eviction rates, ghost-hit counts. |
| `refs_track`                  | `0`           | 0 / 1                       | Collect the access statistics (`refs_stats`, `refs_dist`). Counting itself is always on; this only switches the histograms. |
| `refs_stats` / `refs_dist`    | —             | read; write resets          | Access, dedup, isolation, survivor and refault-distance histograms by REFS. |
| `refs_cap`                    | `7`           | 1–15                        | Saturation of REFS. |
| `epoch_clock` / `epoch_time_ms` | `0` / `1000` | 0 = install, 1 = time; ms  | Epoch clock: by installed pages (default) or by wall time. A comparison aid. |
| `state_probe`                 | —             | PFN, then read              | Write a PFN, read back its decoded state word. |
| `owner_audit`                 | —             | write 1, then read          | Cross-check the owner tags against the folios on a quiescent system (`CONFIG_MEMCG`). |
| `owner_verify`                | `0`           | 0 / 1                       | Make cgroup reclaim look at what its index hid and count it (`CONFIG_MEMCG`). |
| `gen_floor_shift`             | `0`           | 0–6                         | Extra right-shift on the install-cadence floor; a debugging aid. |
| `kcompressd`                  | `24`          | -100..+100                  | Async swap queue depth. `0` disables. Positive = Marie-gated; negative = force-enable regardless of Marie. Present when `CONFIG_SWAP=y`. |
| `defrag`                      | `1`           | 0 / 1                       | Marie defrag replaces stock compaction inside kcompactd. `0` = fall back to stock compaction at runtime. `CONFIG_LRU_MARIE_DEFRAG` only. |
| `defrag_drop`                 | `1`           | 0 / 1                       | Killswitch for just the defrag DROP path (MOVE stays active). `CONFIG_LRU_MARIE_DEFRAG` only. |
| `defrag_stats`                | `0`           | 0 / 1, then read            | Write `1` to enable the cumulative defrag diagnostic, then read the node for it. `CONFIG_LRU_MARIE_DEFRAG` only. |
| `walker_interval_critical_ms` | `HZ/30` (~33 ms) | ms                       | Walker cadence under critical memory pressure. |
| `walker_interval_low_ms`      | `HZ/10` (100 ms) | ms                        | Walker cadence under low pressure. |
| `walker_interval_normal_ms`   | `HZ/4` (250 ms) | ms                         | Walker cadence under normal pressure. |
| `walker_interval_idle_ms`     | `HZ` (1000 ms)  | ms                         | Walker cadence when idle. |

Generation size has no knob — it is derived from live occupancy per type; see [Gen-ring advance policy](#gen-ring-advance-policy-skip-on-full-no-forced-drain). `low_swappiness_mode` is gone: `max_swappiness` replaces it (`1` = the old `low_swappiness_mode=1`, `200` = the old `0`).

Boot cmdline: `lru_marie=0` disables Marie at boot (and `lru_marie=1` forces it on), overriding `CONFIG_LRU_MARIE_DEFAULT_ON` either direction. This is the only way to select Marie vs MGLRU / Legacy — the choice is latched into the static key before any folio is tracked, so the `enabled` node cannot be flipped at runtime. Three more boot-only parameters:

- `lru_marie_max_l2_pages_per_bit=N` — ceiling on PFNs per L2 bit (default 8192, ~32 MiB at 4 KiB pages). Consumed at `subsys_initcall` to size the bitmap and range-lock arrays, so it is deliberately not sysfs-tunable.
- `lru_marie_owner_max=N` — cap on the memcg owner slots handed out (default 65535); a test aid.
- `lru_marie.simd=0|1` and `lru_marie.simd_max=avx512|avx2|sse2` (x86) — seed the SIMD knobs before the first walker pass; overridden by any later sysfs write.

---

## Recommended Configuration

### The swappiness Paradigm: Storage Speed Matters

**On modern hardware (NVMe/SSD and ZRAM) there is nothing to set.** Marie caps its own effective swappiness at 1 in-kernel via `max_swappiness` (default `1`), so whatever value your distro, a udev rule, or a tuning daemon installed in `vm.swappiness` is simply ignored by the reclaim pick driver. Marie does **not** rewrite the kernel's `vm_swappiness` default — it stays at the upstream 60 — because the cap makes that override unnecessary, and leaving the sysctl alone keeps its reported value honest for everything else that reads it.

The cap only ever *lowers*: a deliberate `vm.swappiness=0` — the hard "never swap" contract — is still honoured verbatim.

**The Philosophy for Fast Storage:** Modern operating systems strive to keep physical RAM completely filled with file caches. When even slight memory pressure occurs, the upstream default (`swappiness=60`) dictates that the kernel should proportionally swap out anonymous pages. However, dropping a clean file page is instantaneous, whereas swapping out an anonymous page to ZRAM consumes significant CPU time, pollutes CPU caches with compression codecs, and blocks the calling context — causing visible UI stutter. With fast modern storage (NVMe/SSD), reading back a dropped file cache (refault) takes only microseconds and is virtually unnoticeable. Thus, forcing the system to grind on ZRAM swap-outs during normal daily usage just to preserve cold file caches is a severe waste of resources that degrades the user experience. Marie's `max_swappiness` default encodes exactly this judgement.

**The Philosophy for Slow Storage:** If your system relies on **slower storage like HDDs**, the cost of reading back dropped file caches is high and causes severe unresponsiveness. Here you want the proportional controller instead — which takes **two** steps, because the cap would otherwise flatten whatever you configure:

```
echo 200 > /sys/kernel/mm/lru_marie/max_swappiness      # remove the cap
sysctl -w vm.swappiness=60                              # now honoured verbatim
```

With the cap removed, Marie honours values from 2 to 199 by engaging its proportional controller, balancing eviction between anon and file pages to minimize the severe I/O penalties of slow disks. Setting `vm.swappiness` alone, without raising `max_swappiness`, has no effect for any value above the cap. (A cap such as 10 was tried as the default: its small anon share raised anon refaults by up to an order of magnitude while file was plentiful, and nothing paid for it.)

**The Synergy (at effective swappiness 1):** In Legacy LRU or MGLRU, running at `swappiness=1` is dangerous; it often leads to severe thrashing and livelocks because the kernel will evict the entire file cache (including active working sets) before it starts swapping. Marie LRU completely solves this via the `clean_min_ratio` hard floor (inherited from `le9uo`).

The default pairing of `max_swappiness=1` with `clean_min_ratio=10` gives a robust synergy on modern systems:
1. **During normal use**, the kernel exclusively drops "cheap" file caches. Zero CPU time is wasted on premature anonymous swap-outs.
2. **Under severe pressure**, once the file cache drops to the `clean_min_ratio` floor, Marie securely locks the remaining file working set and forcibly diverts all reclaim pressure to anonymous pages (swap-out).

This guarantees that swap-out is strictly reserved for times of *genuine* memory exhaustion, delivering a vastly smoother desktop experience without the risk of thrashing.

*(Note for specialized servers: For environments that lock massive amounts of memory—such as large databases or virtualization hosts—it is recommended to lower `clean_min_ratio` to `5` or less. This prevents the file cache floor from overly restricting the remaining RAM available to your locked memory workloads.)*

### Disable `systemd-oomd`

It is highly recommended to **disable or mask `systemd-oomd`**. Marie LRU's internal memory management is robust under heavy pressure, and the kernel side converges on "OOM promptly rather than freeze" through the conditions in [When Marie invokes the OOM killer](#when-marie-invokes-the-oom-killer):

1. the `clean_min_ratio` floor (enforced on every reclaim path) lets reclaim reach a clean no-progress state at the floor, so the kernel's own OOM killer fires without thrashing;
2. a swap store that cannot take a page (Capable) and a turnover of evictions that freed no RAM (Productive) are caught directly, without waiting for a stall to build up;
3. the swap meter (Cool) cuts a treadmill that holds up the tasks that want to run, after `oom_heat_limit` seconds of it without a break — the one shape the arithmetic above cannot observe. It is the knob to tune if you want a shorter or longer fuse than the 30 s default.

User-space OOM daemons like `systemd-oomd` often misinterpret Marie's proactive cache management as memory exhaustion and prematurely kill user applications.

⚠️ On very low-core or low-RAM machines a user-space PSI-based OOM daemon is still worth keeping — mechanisms 1–3 are kernel-side and cannot help if the CPUs themselves are saturated in compression before any of them arms (and the meter needs PSI).

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
| `CONFIG_LRU_MARIE` | `y` | `MMU` | Builds Marie. Memory cost ≈4 bytes per RAM PFN for the state word (≈1 MiB per GiB: 16 MiB on 16 GiB) plus ≈0.5 MiB per GiB of scan planes, attached per 128 MiB memory section as RAM goes online. |
| `CONFIG_LRU_MARIE_DEFAULT_ON` | `y` | `LRU_MARIE` | Compile-time default of the boot static key. `N` = built in but off unless `lru_marie=1`. The command line always overrides either direction. |
| `CONFIG_LRU_MARIE_DEFRAG` | `y` | `LRU_MARIE && COMPACTION` | Marie defrag inside kcompactd. `N` compiles every hook out — the image is identical to plain Marie. |

`CONFIG_LRU_GEN` and `CONFIG_LRU_GEN_ENABLED` should stay as they were — Marie does not replace MGLRU at build time, only displaces it at boot (via the `lru_marie=` static key) when enabled.

To verify Marie is active after boot:

```
cat /sys/kernel/mm/lru_marie/enabled       # 1 = active
cat /sys/kernel/mm/lru_marie/version       # e.g., 1.0.0-rc1
```

---

## Status & roadmap

| Area                      | State            | Notes |
| :------------------------ | :--------------- | :------------------------------------------------------------------------------------------------------------------- |
| x86-64 (AVX-512F)         | ✅ working        | preferred SIMD path |
| x86-64 (AVX2 / SSE2)      | ✅ working        | auto-selected when AVX-512F absent |
| ARM64                     | ⚠️ scalar only    | NEON walker pending FPU save/restore profiling vs. the scalar baseline |
| Other arches              | ⚠️ scalar only    | functional, no SIMD acceleration |
| Memory hotplug            | ✅ working        | per-128 MiB-section attach/detach; QEMU-tested with DIMM add/remove, virtio-balloon, virtio-mem |
| Defrag (`CONFIG_LRU_MARIE_DEFRAG`) | ⚠️ prototype | verified safe (refault-free migration, bounded DROP, no corruption); yield-vs-stock A/B not yet run on real hardware |

The `mm/lru_marie/` directory is version-agnostic: every in-tree mm API that changed across the supported kernels is isolated behind uniform names in the headers under `compat/` — `state_compat.h`, `walker_compat.h`, `defrag_compat.h`, `vmap_compat.h` — and `LINUX_VERSION_CODE` is tested in exactly one place, `compat/caps.h`, which turns it into named capabilities. All of `mm/lru_marie/`, compat included, and `include/linux/lru_marie.h` are byte-identical across the supported kernels; what differs per version is the host call sites (`mm/vmscan.c`, `mm/zswap.c`, `mm/workingset.c`, `mm/page_io.c`, `mm/memcontrol.c` and friends). The capabilities and the version each one starts at:

- `post_alloc_hook()` no longer hands the page out with a refcount: **6.14**;
- `shrink_folio_list()` takes the memcg being reclaimed for as a trailing parameter: **6.16**;
- `has_unmovable_pages()` takes an `enum pb_isolate_mode` and `pmd_devmap()` is gone: **6.17**;
- `folio->flags` is the typed `memdesc_flags_t` (`struct { unsigned long f; }`) and `folio_pte_batch()` is replaced by `folio_pte_batch_flags()`: **6.18**;
- `arch_enter/leave_lazy_mmu_mode()` are renamed to `lazy_mmu_mode_enable/disable()`, and `vmap_pages_range_noflush()` takes a gfp mask: **7.0**;
- the batched no-flush young-PTE API `test_and_clear_young_ptes_notify()` is native (compat emulates it as a per-PTE `ptep_clear_young_notify()` loop on earlier bases), and `PGSTEAL_*` / `PGSCAN_*` moved from `enum vm_event_item` to `enum node_stat_item`: **7.1**;
- the buddy allocator's private API moved to `mm/page_alloc.h` and `post_alloc_hook()` gained an `alloc_flags` argument, `mem_cgroup_swappiness()` moved to `mm/swap.h`, and `vmap_pages_range_noflush()` is declared in `mm/vmalloc.h`: **7.3**.

Marie keeps no per-memcg lists or aging state (the 7.1 objcg-reparent handling that earlier per-memcg revisions needed is gone). The only memcg-facing code is the owner slot of cgroup-targeted reclaim, hooked at memcg alloc/free and at charge / move_account. On the 7.x kernels, where a cgroup going offline moves its charged folios to the parent while the cgroup itself is freed, the host also calls `lru_marie_memcg_reparented()`: the dead slot is forwarded to the parent's, one sweep over the state array rewrites the tags, and only then is the slot released for reuse.

### Source layout

`mm/lru_marie/` is layered; each directory is described below:

| Directory | Contents |
| :--- | :--- |
| `model/` | Pure logic with no kernel dependency: state word, the state machine, placement, ring, victim pick, OOM judgement. |
| `store/` | The state array, sections, owner index, bitmaps and counters. `apply.h` is the only writer of a state word. |
| `engine/` | Hooks, install / access, aging, scan, isolate, pressure, walker, OOM driver. |
| `defrag/`, `arch/` | The optional defragmenter; per-architecture SIMD. |
| `compat/` | `caps.h` (the only version tests), `*_compat.h`. |
| `sysfs.c`, `init.c` | Tunables; boot. |


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
