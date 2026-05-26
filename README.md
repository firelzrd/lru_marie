# Marie LRU

**Multi-graded Adaptive Reclaim & Independent Eviction**

Marie is an out-of-tree, third-pillar LRU implementation for the Linux kernel, sitting alongside Legacy LRU and MGLRU rather than replacing either.  It composes three established ideas — working-set protection ([le9uo](https://github.com/firelzrd/le9uo)), proportional anon/file aging under a generational ring ([Re-swappiness](https://github.com/firelzrd/re-swappiness)), and async swap-out offload ([kcompressd-unofficial](https://github.com/firelzrd/kcompressd-unofficial)) — onto a redesigned core: per-lruvec lock-free pending queues, a bounded generation ring without forced oldest-gen drain, and a SIMD-accelerated PTE walker driven by rmap-to-walker bloom-filter feedback.

Marie's paths are gated behind a runtime static key (`lru_marie_enabled()`), so a kernel built with `CONFIG_LRU_MARIE=y` but Marie turned off behaves exactly as MGLRU / Legacy LRU would.

---

## Background

Linux's page-reclaim LRU has gone through two main eras.

**Legacy LRU** (the upstream default before MGLRU, and still the fallback) maintains per-memcg active/inactive list pairs.  Folios are promoted on `PG_referenced` and demoted on scan-tail age-out.  The model is small and cheap, but the binary active/inactive boundary is coarse — within each bucket there is no further ordering, so when the inactive list isn't large enough to span the working set, refault-driven re-promotion turns into a thrash treadmill.

**MGLRU** (`lru_gen`, mainlined in 6.1) replaced the two-bucket model with a generational ring, actively harvested by a PTE walker rather than relying solely on rmap-side `PG_referenced`.  This delivered a large improvement on workloads where the active/inactive cut sliced through the working set, and remains the right baseline for most general-purpose use.

Marie is a third design point.  The axes where MGLRU's choices turn into trade-offs under specific workloads — and where Marie picks the other end:

Legend — ✅ behaves as expected · ⚠️ works but with a structural trade-off · ❌ not provided / cannot deliver the property

| Aspect                | Legacy LRU                                                                | MGLRU                                                                                                       | Marie                                                                                                       |
| :-------------------- | :------------------------------------------------------------------------ | :---------------------------------------------------------------------------------------------------------- | :---------------------------------------------------------------------------------------------------------- |
| Aging granularity     | ⚠️ binary active / inactive only                                           | ✅ 4-gen ring (`MAX_NR_GENS=4`); a full ring triggers a forced `inc_min_seq` oldest-gen drain                 | ✅ 4-gen ring (`MARIE_MAX_NR_GENS=4`); advance skips when the ring is full, no forced drain                  |
| anon/file iteration   | ✅ per-type lists; `vm.swappiness` works as a proportional anon:file ratio | ❌ shared `max_seq` across anon/file reduces `vm.swappiness` to a per-generation hint                        | ✅ per-type `seq` / `min_seq` / ring; restores `vm.swappiness` as a proportional ratio                       |
| Hot-signal harvest    | ⚠️ rmap-driven `PG_referenced` only; no active walker                      | ✅ PTE walker + walker-internal PMD bloom filter + rmap look-around cascading `PG_referenced`                | ✅ Vectorized PTE walker + rmap-fed PMD bloom filter + rmap look-around updating a saturating tier bit only            |
| Compression stall     | ❌ kswapd blocks inside `zswap_store`                                      | ❌ kswapd blocks inside `zswap_store`                                                                        | ✅ offloaded to per-node `kcompmari` kthread                                                                 |
| Clean-file floor      | ❌ none                                                                    | ❌ none                                                                                                      | ✅ `clean_min_ratio` hard floor against thrashing                                                            |
| Reclaim-livelock OOM  | ❌ retries until `MAX_RECLAIM_RETRIES`                                     | ❌ retries until `MAX_RECLAIM_RETRIES`                                                                       | ✅ `clean_min_ratio` floor enforced on every reclaim path → reclaim stops cleanly at the floor and the stock no-progress OOM fires promptly; plus a swap-write-failure fast-path |

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
                   residency ADT,
                   bounded gen ring w/o
                   forced inc_min_seq,
                   SIMD PTE walker,
                   rmap-fed bloom)
```

Marie keeps the **goals** of its predecessors but doesn't reuse their implementations verbatim — it re-derives each idea inside its own LRU core so the surrounding state (gen ring, lock topology, scan order) is internally consistent.

- From **le9uo**: hard working-set protection (`clean_min_ratio`).  Like le9uo, Marie enforces the floor on the single path that actually reclaims and then lets the kernel's stock no-progress OOM path fire — once clean file is at the floor and anon is unreclaimable, reclaim returns no progress and the OOM killer fires promptly, with no separate early-OOM watermark needed (this is why le9uo converges at any floor size).  `anon_min_ratio` and `clean_low_ratio` from le9uo are intentionally **not** ported — under Marie the former is largely redundant with `vm.swappiness` once `kcompmari` brings anon refault sub-millisecond, and the latter overlaps with the existing `swap_tokens` bucket.
- From **Re-swappiness**: per-type `seq` / `min_seq` so `vm.swappiness` acts as a proportional anon:file scan ratio under a generational ring (the property Legacy LRU already has and MGLRU loses by sharing `max_seq`).  Re-swappiness ports this onto MGLRU's structure; Marie owns its own per-type state inside `marie_lruvec` and does not depend on MGLRU being per-type.
- From **kcompressd-unofficial**: per-node async compression FIFO so kswapd doesn't block in `zswap_store` / `__swap_writepage`.  Marie carries this over as `kcompmari` (renamed from `kcompressd` for Marie's namespace) with the same per-node FIFO design and the same 0–256 depth knob, default 24.

---

## Architecture

```
                 fault path
                      │
                      ▼
             ┌──────────────────────┐
             │ marie_state[pfn]     │  flat 1-byte per-PFN array
             │  (TRACKED, TYPE,     │  (no linked lists, no staging)
             │   ZONE, GEN, TIER)   │
             └──────────┬───────────┘
                        ▲
   rmap look-around     │   tier++ on young PTE
   tier++ (cmpxchg)     │   (SIMD scan)
           │            │           │
           ▼            ▼           ▼
   ┌────────────────────────────────────────┐
   │        L1 / L2 tracking bitmaps        │
   │  global (type, gen, tier) + per-memcg  │
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
             │ kcompmari kthread    │
             │  per-node FIFO       │
             │  zswap_store /       │
             │  __swap_writepage    │
             └──────────────────────┘
```

- **Flat per-PFN state array (`marie_state`)**: Marie abandons linked lists entirely. Every folio's state is encoded in a single byte (TRACKED, TYPE, ZONE, GEN, TIER) within a flat array allocated once at boot. This eliminates allocation in the fast paths and per-CPU staging queues.
- **L1/L2 tracking bitmaps**: To efficiently scan the flat array, Marie maintains 512-bit summarized L2 bitmaps for both global (type, gen, tier) combinations and per-memcg accounting, allowing the isolate scanner to skip empty 32 MiB ranges in a single cycle.
- **Independent per-type rings**: Aging is tracked by `marie_head_gen[type]` cycling counters. Anon and file have fully independent rings, restoring `vm.swappiness` as a proportional scan ratio without forcing oldest-gen drains.
- Hot-signal pickup is split across rmap and the walker. MGLRU also uses a bloom filter, but it's walker-internal — a self-feedback to skip PMDs that the walker itself did not touch on the previous pass. Marie's bloom is cross-component: rmap is the producer (the PTL-bounded look-around flags PMDs that just took a young hit), the walker is the consumer (it pays the SIMD scan + tier++ cost only on bloom-hit PMDs). This lets a hot signal observed by rmap influence the walker within the same reclaim window, not the next one.

---

## Design

### Reclaim pipeline: Flat per-PFN state array

Marie tracks a folio's lifecycle entirely via a single byte per PFN in `marie_state[pfn]`. The 8 bits encode:

- **1 bit TRACKED**: 1 = owned by Marie, 0 = untracked.
- **1 bit TYPE**: anon or file.
- **2 bits ZONE**: `folio_zonenum`.
- **2 bits GEN**: relative position (0..3) in the cycling ring.
- **2 bits TIER**: saturating hotness counter (0..3), bumped by the walker or rmap.

Because state is purely array-based, there are no `list_del` or `swap-pop` operations during eviction or installation. `marie_state[pfn] = 0` instantly drops tracking. To isolate folios, Marie uses an **L2-pruned `__ffs`/`blsr` walk** combined with **Two-Stage Software Prefetching**:
- **L2-pruning**: Bitwise ANDing the global (gen, tier) L2 bitmap with the memcg's L2 bitmap skips empty 32 MiB chunks in 1 CPU cycle.
- **Hardware Extraction**: Surviving L2 blocks are resolved to precise PFNs in near O(1) time using `__ffs` (Find First Set) and `blsr` (reset lowest set bit) hardware instructions over the L1 bitmap.
- **Strategic Prefetch**: A "Cache-line cursor" tracks boundaries to issue exactly one prefetch per cache-line. It fires `prefetcht2` (DRAM → L3) far ahead of the processing cursor, and `prefetcht0` (L3 → L1) slightly ahead, hiding the DRAM latency wall without thrashing the L1 cache.

### Gen-ring advance policy: skip on full, no forced drain

MGLRU's ring has `MAX_NR_GENS=4`.  When it fills, aging calls `try_to_inc_min_seq()` to push the oldest gen forward before cutting a new head.  If that oldest gen holds folios that won't move — clean file pages under a working-set policy, mlocked anon, etc. — aging still demands forward progress, so the same untouchable folios get rewalked over and over: the *forced `inc_min_seq` treadmill*.

Marie uses a 4-gen cap (`MARIE_MAX_NR_GENS=4`) tracked by `marie_head_gen[type]`. Head-generation advance is *drain-wait gated*: `marie_try_advance_head()` will not advance if the next slot in the ring still contains folios.

Practical effect: an oldest gen full of `clean_min_ratio`-protected folios is left alone.  Aging stops growing new heads until reclaim drains the existing tail (or `clean_min_ratio` diverts the type selector to anon).  The treadmill never starts because nothing in Marie demands aging-side progress regardless of consumer state.

### Independent anon/file pipelines

Marie keeps **fully independent** per-type state — each of ANON and FILE has its own `marie_head_gen[type]` cycling counter and tracking bitmaps. There is no shared iteration counter between the two types, so `vm.swappiness` works as a proportional anon:file scan ratio — the behaviour Legacy LRU has always had, recovered inside a generational ring.

### Hot-signal harvest

- **SIMD PTE walker** (per-pgdat).  On x86-64, `arch_initcall` picks the widest available SIMD instruction set (AVX-512F > AVX2 > SSE2) and flips a static branch so the walker's young-bit extraction has no indirect call. The walker uses SIMD to read 512 64-bit PTEs at once and extract their Accessed bits via vector masks, entirely avoiding per-PTE branching. ARM64 and other arches use a scalar fallback for now.
- **FPU batching**.  `kernel_fpu_begin/end` are amortised across `MARIE_FPU_BATCH=16` consecutive bloom-hit PMDs, then released around the `walk_page_range` boundary so the preempt-disabled window stays bounded by `MARIE_FPU_BATCH`, not by walk length.
- **rmap look-around**.  Called from `folio_referenced_one()`, this scans up to `BITS_PER_LONG` PTEs around the target folio's PMD under the existing PTL.  Unlike MGLRU's look-around it does **not** set `PG_referenced` on neighbouring folios — that cascade is the main source of reclaim starvation under fault-heavy workloads.  Instead it feeds the walker via a bloom filter (below).
- **rmap-fed PMD bloom feedback**.  MGLRU's bloom filter is walker-internal: the walker remembers which PMDs it touched last pass and uses that to skip cold PMDs next pass.  Marie's bloom is cross-component — rmap is the producer ("this PMD had a young PTE on its target"), the walker is the consumer ("only scan PMDs the bloom flagged").  The filter is double-buffered and swapped at the end of each walker pass, so each pass acts on the rmap signal accumulated during the previous reclaim window.
- **Pressure-adaptive cadence**.  The walker re-evaluates its scan interval (`walker_interval_{critical,low,normal,idle}_ms`) on each pass.  Defaults: `HZ/30`, `HZ/10`, `HZ/4`, `HZ` — all tunable.

### Pressure resilience

- **`clean_min_ratio` (hard working-set floor).**  Marie withholds file reclaim — on *every* reclaim path (the per-PFN pick driver and the legacy drain that follows it in `shrink_lruvec`) — when clean file pages would otherwise be evicted below the configured percentage of node RAM, diverting reclaim to anon instead. Only clean file counts toward the floor: dirty pages, which cannot be reclaimed without writeback, are subtracted so they cannot satisfy the floor on unreclaimable pages. Equivalent in spirit to le9uo's knob of the same name.  Default 10%; set to 0 to disable.

  The pick driver selects the reclaim type via a strict priority cascade: OOM victims and anon-unreclaimable conditions force FILE-only (no point swapping if no slots exist or the reaper is taking the anon anyway); `vm.swappiness=0` forces FILE-only regardless of the floor (the hard no-swap contract — at the floor file is also blocked, so this OOMs rather than swaps, which is the intended behaviour); the floor-in-force condition forces ANON-only, outranking the proportional controller so a FILE-biased pick cannot get pinned on the floor-blocked side and stall reclaim at high `vm.swappiness` values. Otherwise the swap-bias proportional pick applies.

  The legacy orphan drain in `shrink_lruvec()` that follows the picker (draining folios that landed on `lruvec->lists` due to failed Marie installs or drain handoffs) mirrors the pick result via a `MARIE_DRAIN_*` bitmask returned by `lru_marie_shrink_lruvec()`. The drain zeroes `nr[]` for any type the picker did not scan, preventing `get_scan_count()`'s `SCAN_EQUAL` path (triggered at `sc->priority == 0`) from evicting file behind the picker's back during swap-fill at high `vm.swappiness`.
- **No-progress OOM + swap-write-failure fast-path.**  Because the floor is enforced on every reclaim path, once clean file is at the floor and anon is unreclaimable reclaim returns no progress — so the kernel's stock `no_progress_loops` path in `should_reclaim_retry()` reaches the OOM killer promptly, with no separate free-memory watermark needed (this is exactly how le9uo converges at any floor size).  One Marie-specific addition remains in that slowpath: it also aborts the retry loop as soon as the swap backend has rejected more than `MAX_SWAP_WRITE_FAIL_RETRIES` writes during this allocation attempt — e.g. ZRAM `zs_malloc` starvation, where `can_reclaim_anon_pages()` still reports free slots that no longer accept writes, a case the floor alone would only resolve after grinding file down to it.
- **`kcompmari` async swap-out.**  Per-node kernel thread that drains a `kfifo` of anon folios deposited by kswapd, running `zswap_store` / `__swap_writepage` off the reclaim critical path. The depth knob takes signed values from `-100` to `+100` (default `+24`). `0` disables the offload, positive values queue up to `|v|` folios and are gated by Marie's enabled state, and negative values force-enable the thread even when Marie is turned off.

### Coexistence

- **Static-key gate.**  Every Marie entry point is fronted by `lru_marie_enabled()`, a `DEFINE_STATIC_KEY_TRUE`. When Marie is runtime-disabled the branch is patched out, so MGLRU / Legacy users see zero added instructions on the hot paths.
- **No folio->flags footprint.** Unlike 0.1.0, Marie 0.2.0 no longer reserves bits in `folio->flags`. All tracking state resides entirely inside the `marie_state` array, making its memory layout completely independent of core MM flags.

---

## Tunables

All knobs live under `/sys/kernel/mm/lru_marie/` and accept runtime writes (except for the read-only `version` and `stats` nodes).

| Knob                          | Default       | Range / unit                | What it does                                                                                                 |
| :---------------------------- | :------------ | :-------------------------- | :----------------------------------------------------------------------------------------------------------- |
| `enabled`                     | `1`           | 0 / 1, static key           | Master gate.  `0` patches Marie out of the hot paths; MGLRU / Legacy take over.                              |
| `simd`                        | `1`           | 0 / 1, static key           | Toggles SIMD young-bit scan.  `0` falls back to scalar (A/B testing).                                        |
| `version`                     | —             | read-only                   | Reports the current Marie LRU version.                                                                       |
| `stats`                       | —             | read-only                   | Exposes internal reclaim and walker statistics.                                                              |
| `clean_min_ratio`             | `10`          | 0–100, %RAM                 | Hard floor on clean file pages.  Below this, reclaim diverts to anon instead of evicting file.               |
| `kcompmari`                   | `24`          | -100..+100                  | Async swap queue depth. `0` disables. Positive = Marie-gated; negative = force-enable regardless of Marie. |
| `gen_growth_threshold`        | `8192`        | folios (`SWAP_CLUSTER_MAX << 8`) | Head-generation growth limit before a new gen is cut.                                                   |
| `walker_interval_critical_ms` | `HZ/30` (~33 ms) | ms                       | Walker cadence under critical memory pressure.                                                               |
| `walker_interval_low_ms`      | `HZ/10` (100 ms) | ms                        | Walker cadence under low pressure.                                                                           |
| `walker_interval_normal_ms`   | `HZ/4` (250 ms) | ms                         | Walker cadence under normal pressure.                                                                        |
| `walker_interval_idle_ms`     | `HZ` (1000 ms)  | ms                         | Walker cadence when idle.                                                                                    |

Boot cmdline: `lru_marie=0` disables Marie at boot (equivalent to writing `0` to `enabled` immediately after boot).

---

## Recommended Configuration

### The `vm.swappiness=1` Paradigm

For desktop and general-purpose workloads, **`vm.swappiness=1` is the strongly recommended setting** when using Marie LRU.

**The Philosophy:** Modern operating systems strive to keep physical RAM completely filled with file caches. When even slight memory pressure occurs, default settings (like `swappiness=60`) dictate that the kernel should begin swapping out anonymous pages. However, dropping a clean file page is instantaneous, whereas swapping out an anonymous page consumes significant CPU time and I/O bandwidth. With fast modern storage (NVMe/SSD), reading back a dropped file cache is incredibly "cheap". Thus, forcing the CPU to grind on swap-outs during normal daily usage just to load a new application is a severe waste of resources that degrades the user experience.

**The Synergy:** In Legacy LRU or MGLRU, setting `swappiness=1` is dangerous; it often leads to severe thrashing and livelocks because the kernel will evict the entire file cache (including active working sets) before it starts swapping. Marie LRU completely solves this via the `clean_min_ratio` hard floor (inherited from `le9uo`). 

By combining `vm.swappiness=1` with `clean_min_ratio=10` (the default), you achieve a robust synergy:
1. **During normal use**, the kernel exclusively drops "cheap" file caches. Zero CPU time is wasted on premature anonymous swap-outs.
2. **Under severe pressure**, once the file cache drops to the `clean_min_ratio` floor, Marie securely locks the remaining file working set and forcibly diverts all reclaim pressure to anonymous pages (swap-out).

This guarantees that swap-out is strictly reserved for times of *genuine* memory exhaustion, delivering a vastly smoother desktop experience without the risk of thrashing.

*(Note for specialized servers: For environments that lock massive amounts of memory—such as large databases or virtualization hosts—it is recommended to lower `clean_min_ratio` to `5` or less. This prevents the file cache floor from overly restricting the remaining RAM available to your locked memory workloads.)*

### Disable `systemd-oomd`

It is highly recommended to **disable or mask `systemd-oomd`**. Marie LRU's internal memory management is extremely robust under heavy pressure. The `clean_min_ratio` floor (enforced on every reclaim path) lets reclaim reach a clean no-progress state at the floor, so the kernel's own OOM killer fires promptly without thrashing, and a swap-write-failure fast-path deep inside the allocator slowpath (`should_reclaim_retry()`) catches ZRAM stalls instantly. User-space OOM daemons like `systemd-oomd` often misinterpret Marie's proactive cache management as memory exhaustion and prematurely kill user applications.

---

## Build & install

To apply against a matching source tree:

```
cd /path/to/linux
patch -p1 < lru_marie.patch
make olddefconfig    # answer Y to CONFIG_LRU_MARIE
make -j$(nproc)
```

`CONFIG_LRU_MARIE` defaults to `y` (Marie depends on `CONFIG_MMU`). `CONFIG_LRU_GEN` and `CONFIG_LRU_GEN_ENABLED` should stay as they were — Marie does not replace MGLRU at build time, only at runtime.

To verify Marie is active after boot:

```
cat /sys/kernel/mm/lru_marie/enabled       # 1 = active
cat /sys/kernel/mm/lru_marie/version       # e.g., 0.2.0
```

---

## Status & roadmap

| Area                      | State            | Notes                                                                                                                |
| :------------------------ | :--------------- | :------------------------------------------------------------------------------------------------------------------- |
| x86-64 (AVX-512F)         | ✅ working        | preferred SIMD path                                                                                                  |
| x86-64 (AVX2 / SSE2)      | ✅ working        | auto-selected when AVX-512F absent                                                                                   |
| ARM64                     | ⚠️ scalar only    | NEON walker pending FPU save/restore profiling vs. the scalar baseline                                               |
| Other arches              | ⚠️ scalar only    | functional, no SIMD acceleration                                                                                     |

Known kernel bases with Marie 0.2.0 patches in this repo: 6.12.74, 6.18.22, 7.0, 7.1-rc1.  All four ports share the same Marie source tree (`mm/lru_marie*.c/.h`, `include/linux/lru_marie*.h`) — the patches differ only in base-kernel-side adaptations.  In particular, the batched no-flush young-PTE API used by Marie's walker is native on 7.1-rc1 (`test_and_clear_young_ptes` / `test_and_clear_young_ptes_notify`), absent on 7.0 (the 7.0 patch back-ports it into `include/linux/pgtable.h` and `include/linux/mmu_notifier.h`), and absent on 6.18 and 6.12 (the 6.18 and 6.12 patches back-port the same API plus thin `lazy_mmu_mode_enable/disable` wrappers, since they only have `arch_enter_lazy_mmu_mode` and lack the 7.0 reentrancy-tracking helpers).

---

## Acknowledgements

Marie LRU builds upon the foundational ideas and pioneering work of several developers in the Linux memory management community. We express our deepest gratitude and respect to:

- **Yu Zhao (Google)**: The original author of **MGLRU (Multi-Gen LRU)**. MGLRU revolutionized Linux page reclaim by introducing the generational ring, the active PTE walker, and the PMD Bloom filter. Marie LRU's foundational concepts — specifically generational aging, background page table scanning, and Bloom filter-driven hot-signal harvesting — are profoundly influenced by MGLRU's trailblazing design.
- **Alexey Avramov (a.k.a. hakavlad)**: The original author of [le9](https://github.com/hakavlad/le9-patch). Marie's `clean_min_ratio` working-set protection and pressure-resilience philosophies are the direct spiritual successors to the le9 project.
- **Qun-Wei Lin (MediaTek)**: The original author of the **Kcompressd** concept. Marie's `kcompmari` asynchronous swap-out offload is heavily inspired by their architectural design to decouple reclaim from `zswap_store`.

### Predecessor patches

[firelzrd/le9uo](https://github.com/firelzrd/le9uo)
[firelzrd/Re-swappiness](https://github.com/firelzrd/re-swappiness)
[firelzrd/kcompressd-unofficial](https://github.com/firelzrd/kcompressd-unofficial)

## License

GPL-2.0, same as the Linux kernel.
