# linux-native

A custom Arch Linux kernel package based on [linux-zen](https://github.com/zen-kernel/zen-kernel),
tuned for compile throughput on an Intel Core Ultra 7 155H (Meteor Lake, hybrid P+E core architecture).

---

## What Changed From linux-zen

### PKGBUILD Changes

**Package renamed: `linux-zen` -> `linux-native`**

The upstream `linux-zen` PKGBUILD uses `pkgver` for both the package version and
to derive source URLs. Since we want a custom version string (`6.19.6.native1`) while
still tracking the upstream zen release (`6.19.6.zen1`), a separate `_pkgver` variable
was introduced:

```bash
pkgbase=linux-native
pkgver=6.19.6.native1   # our package version
_pkgver=6.19.6.zen1     # upstream zen version (used for source URLs)
```

This lets pacman track our custom build independently while still fetching the correct
upstream sources.

**Build flags: `-march=native -mtune=native -pipe`**

```bash
make -j 2 all KCFLAGS='-march=native -mtune=native -pipe'
```

- `-march=native` — tells GCC to emit instructions for the exact CPU running the build.
  On Meteor Lake this unlocks AVX2, AVX-VNNI, and other ISA extensions that a generic
  `x86_64` build leaves on the table. The resulting kernel binary is NOT portable to
  other machines — that's intentional.
- `-mtune=native` — optimizes instruction scheduling and latency for this specific
  microarchitecture, even for instructions that were already legal under `-march`.
- `-pipe` — uses pipes instead of temp files between compiler stages. Faster on systems
  with enough RAM, avoids unnecessary disk I/O.
- `-j 2` — limits parallel jobs. Counterintuitive, but during a kernel build the linker
  and other serial phases can thrash if you throw all 22 threads at it. Tune this to taste.

**htmldocs disabled**

```bash
#make htmldocs SPHINXOPTS=-QT
```

Kernel HTML documentation generation is slow and irrelevant for a personal build.
Commented out to save significant build time.

---

### Kernel Config Changes (`config.x86_64`)

#### 1. Preemption Model: `PREEMPT` -> `PREEMPT_VOLUNTARY`

```
CONFIG_PREEMPT_VOLUNTARY=y
```

This is the most impactful change for compile throughput. To understand why, you need
to understand what kernel preemption actually does.

**What is kernel preemption?**

When a process is running in kernel space (e.g., inside a syscall), preemption controls
whether the scheduler can forcibly kick it out mid-execution to run something else.

There are three main models:

| Model | Behavior | Latency | Throughput |
|---|---|---|---|
| `PREEMPT_NONE` | Only preempts at explicit yield points (syscall return, interrupt return) | High | Best |
| `PREEMPT_VOLUNTARY` | Adds voluntary preemption points throughout the kernel | Medium | Good |
| `PREEMPT` (full) | Can preempt almost anywhere in kernel code | Low | Worse |
| `PREEMPT_RT` | Real-time, preemptible everywhere including interrupt handlers | Lowest | Worst |

**Why does this matter for compilation?**

A `make -j N` build spawns N compiler processes. Each process spends most of its time
in CPU-bound work — parsing, optimizing, emitting code. With `PREEMPT` (full), the
scheduler aggressively interrupts these processes to maintain low latency for interactive
tasks. Every interruption means:

1. Save the current task's register state
2. Run the scheduler to pick the next task
3. Restore the new task's register state
4. Potentially invalidate CPU caches (especially L1/L2)

With `PREEMPT_VOLUNTARY`, the kernel only yields at explicit checkpoints. Compiler
processes run longer uninterrupted, CPU caches stay warm, and you get better throughput.

Full `PREEMPT` was designed for desktop responsiveness — keeping the UI smooth while
background work runs. For a dedicated compile workload, it's overhead you don't need.

**What about `PREEMPT_DYNAMIC`?**

Your config also has `CONFIG_PREEMPT_DYNAMIC=y`. This allows switching the preemption
model at runtime via the `preempt=` kernel parameter without rebuilding. With
`PREEMPT_VOLUNTARY` as the compiled default, you can still boot with `preempt=full`
if you want the old behavior temporarily.

---

#### 2. Timer Frequency: `HZ=1000` -> `HZ=300`

```
CONFIG_HZ_300=y
CONFIG_HZ=300
```

**What is HZ?**

`CONFIG_HZ` sets the kernel timer tick frequency — how many times per second the
kernel's periodic timer fires. Each tick:

- Runs the scheduler to check if the current task should be preempted
- Updates timers, accounting, and statistics
- Handles time-based events

**The throughput math**

At `HZ=1000`, the timer fires 1000 times/second. At `HZ=300`, it fires 300 times/second.
That's ~700 fewer interrupts per second per CPU core. On a 22-thread system doing a
parallel build, that's potentially 15,400 fewer timer interrupts per second system-wide.

Each interrupt:
- Saves/restores registers
- Runs interrupt handler code
- May trigger a scheduler decision
- Pollutes instruction and data caches

For workloads with many short-lived processes (exactly what `make -j N` produces),
this adds up. Lower HZ means compiler processes get longer uninterrupted time slices.

**Why 300 and not 100 or 250?**

- `HZ=100` — server default, maximum throughput, but noticeably laggy for any interactive use
- `HZ=250` — common desktop compromise
- `HZ=300` — divides evenly into common video frame rates (30fps, 60fps), good balance
- `HZ=1000` — desktop/gaming default, optimized for latency over throughput

300 is the sweet spot: meaningfully less overhead than 1000, still responsive enough
for normal desktop use between builds.

**Note on `NO_HZ` (tickless kernel)**

The kernel also has `CONFIG_NO_HZ_IDLE` which stops the timer tick entirely when a
CPU is idle. This is separate from `HZ` — `HZ` controls the rate when CPUs are busy,
`NO_HZ` eliminates ticks when they're idle. Both can be active simultaneously.

---

## The Linux Scheduler (CFS and Beyond)

Since we're here, a deeper look at what's actually scheduling your compiler processes.

### Completely Fair Scheduler (CFS)

The default Linux scheduler since 2.6.23. The core idea: every runnable task should
get an equal share of CPU time, weighted by priority (nice value).

CFS tracks `vruntime` (virtual runtime) per task — a normalized measure of how much
CPU time a task has consumed. The scheduler always picks the task with the lowest
`vruntime` (the "most deserving" task). This is implemented as a red-black tree ordered
by `vruntime`, so picking the next task is O(log n).

Key tunables (in `/proc/sys/kernel/`):
- `sched_latency_ns` — target scheduling period. All runnable tasks get one turn within this window.
- `sched_min_granularity_ns` — minimum time a task runs before it can be preempted.
- `sched_wakeup_granularity_ns` — how much a waking task's vruntime must be less than the current task's to trigger a preemption.

For throughput, you'd increase `sched_min_granularity_ns` and `sched_latency_ns` to
give tasks longer time slices. This is essentially what `PREEMPT_VOLUNTARY` + lower
`HZ` achieves at the kernel level.

### sched-ext (SCX) — The Future

Your kernel has `CONFIG_SCHED_CLASS_EXT=y`. This is a framework that lets you replace
the scheduler with a BPF program at runtime — no kernel rebuild required.

```bash
# Switch to rusty scheduler (good for hybrid CPUs, throughput-oriented)
systemctl start scx  # uses whatever SCX_SCHEDULER is set in /etc/default/scx

# Or manually
scx_rusty &
```

Available schedulers in `scx-scheds`:
- `scx_rusty` — topology-aware, good for hybrid P+E core CPUs, general throughput
- `scx_lavd` — latency-aware virtual deadline, excellent for Meteor Lake's asymmetric cores
- `scx_bpfland` — interactive-first, prioritizes responsiveness
- `scx_nest` — packs tasks onto fewer cores to improve cache locality (good for compile)

For your use case (`scx_nest` or `scx_rusty` are worth benchmarking against vanilla CFS
with these config changes).

### Hybrid CPU Topology (Your Meteor Lake)

Your Ultra 7 155H has three tiers:
- 6 P-cores (Performance) — high IPC, high frequency, hyperthreaded (12 threads)
- 8 E-cores (Efficiency) — lower IPC, lower power, no HT (8 threads)
- 2 LP-E-cores (Low Power E-cores) — barely worth scheduling on for compile work

`CONFIG_SCHED_MC_PRIO=y` tells CFS to prefer P-cores for tasks and use E-cores as
overflow. This is important — without it, the scheduler treats all cores as equal and
you can end up with compiler processes on LP-E-cores while P-cores sit idle.

If you disable E-cores in BIOS as you mentioned, you eliminate the scheduling complexity
entirely. The scheduler no longer has to make asymmetric decisions, and all 12 threads
(6 P-cores x 2 HT) are equivalent. For pure compile throughput this is often a win
because the scheduler stops second-guessing itself about core placement.

---

## Benchmarking

To measure the impact of these changes, time a kernel build before and after:

```bash
# In the linux source tree
time make -j$(nproc) allmodconfig && time make -j$(nproc)
```

Or more practically, time your own package build:

```bash
time makepkg -s --noconfirm
```

Compare with `perf stat` for deeper insight:

```bash
perf stat -e context-switches,cpu-migrations,cache-misses make -j$(nproc)
```

Lower `context-switches` and `cache-misses` = the config changes are working.
