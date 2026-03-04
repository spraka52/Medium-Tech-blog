# Could Hashing Replace FIFO and Round Robin?

## A systems-level investigation into hash-based CPU scheduling — where it already exists in disguise, why it hasn't been formalized as a primary policy, and whether it should be.

---

If you're systems-programming adjacent, you know the story. The CPU scheduler's job is to answer one question, thousands of times per second: **which runnable process gets the core next?** Every policy is really just a different answer to that question.

FIFO answers it with arrival time. Round Robin answers it with a time-slice carousel. CFS answers it with a virtual runtime red-black tree. Priority queues answer it with a static or dynamic weight. Lottery scheduling answers it with a random draw.

> Every scheduling policy is, at its core, a mapping function: f(process_state) → dispatch_order. The policy differs only in what attributes of process state are fed in and how they're mapped to an ordering.

This framing is important, because it reveals something subtle: a **hash function** is precisely a mapping from an arbitrary key-space to a bounded output space — deterministic, uniform (ideally), and fast. So why has no one seriously proposed it as the primary dispatch mechanism?

Let's find out.

---

# 1. Where Hashing Already Lives in the Kernel

Before proposing anything new, it's worth acknowledging that hashing is not alien to the kernel — it's just never held the scheduler's wheel. Here's where it quietly operates today.

## PID Hash Tables

Both Linux and Windows NT use hash tables for process lookup. When the scheduler resolves a PID to a `task_struct`, it does a hash lookup on `pidhash[]`. This is hashing *in service of* scheduling, but not as the ordering mechanism itself.

## Receive-Side Scaling (RSS)

This is the closest existing analog. RSS is a NIC feature — now also implemented in software as RPS — that hashes incoming packet 4-tuples `(src_ip, dst_ip, src_port, dst_port)` and uses the result to assign interrupts to CPU cores.

```
Incoming Packet
  └── Hash( src_ip || dst_ip || src_port || dst_port )
        └── h % num_cpus  ──→  CPU N handles interrupt
```

This *is* hash-based scheduling of I/O work to processors. It's deterministic, load-distributing, and cache-locality-friendly for connection-affine workloads. The same connection always lands on the same core — a property we'll revisit.

## Consistent Hashing in Distributed Schedulers

In cluster schedulers (Kubernetes, YARN, Mesos), consistent hashing assigns jobs to worker nodes in a way that minimizes remapping when nodes join or leave. Rendezvous hashing (HRW) extends this. These operate above the OS boundary but represent exactly the pattern we're discussing — just at a larger scale.

## SMP Core Affinity via Hash Pinning

Some NUMA-aware runtimes (Go's goroutine scheduler, certain JVM implementations) hash goroutine/thread IDs to preferred OS threads to reduce cross-core migrations. Again: hashing used for locality, not ordering.

---

# 2. The Novel Idea: Hash-Keyed Scheduling as a First-Class Policy

> What if the scheduler computed a hash over composable process attributes and used that hash — directly — to determine dispatch order or CPU slot assignment?

More precisely: given a set of runnable processes *P*, define a hash-key *K(p)* for each process from a combination of its attributes, and schedule based on `H(K(p))`.

## What Would You Hash?

The hash key is composable. You choose which attributes to include based on what scheduling properties you care about:

**`pid` only** → Pseudo-random but deterministic ordering per pid *(analogous to: Lottery, deterministic)*

**`pid + arrival_time`** → Consistent ordering, arrival-sensitive *(analogous to: FIFO-ish, without a queue)*

**`pid + tick_count`** → Rotates priority over time *(analogous to: Round Robin-ish)*

**`priority + burst_estimate`** → Groups processes by weight class *(analogous to: Multilevel queue)*

**`memory_region + pid`** → Cache-locality-aware assignment *(analogous to: NUMA-aware scheduling)*

## Variant A: Hash as Dispatch Rank

The simplest variant: at each scheduling point, compute `H(K(p))` for all runnable processes and dispatch the one with the *minimum* hash value. No queue maintenance, no tree rebalancing.

```c
func hash_schedule(runqueue []) → process:
    best   = null
    best_h = MAX_UINT64

    for p in runqueue:
        // K = composite key: pid XOR (arrival_ns >> 10) XOR priority_weight
        K = p.pid ^ (p.arrival_ns >>> 10) ^ p.weight
        h = xxhash64(K, seed=EPOCH_SEED)

        if h < best_h:
            best_h = h
            best   = p

    return best
```

By changing the `seed` each epoch (e.g., per scheduler tick or per time quantum), the ranking shuffles — giving you a tunable blend of **determinism and rotation**. Freeze the seed → consistent priority. Rotate the seed → fairness over time.

## Variant B: Hash-Bucketed Time Slots

A more structural variant: divide scheduler time into *N* slots per epoch. Each process is assigned a slot by `H(K(p)) % N`. Within a slot, ties are broken by arrival. This gives you a *hash-partitioned time-sharing* model.

```
Epoch (T ms)  subdivided into N slots
─────────────────────────────────────────────────
 Slot 0   │  Slot 1  │  Slot 2  │  ... │  Slot N-1
─────────────────────────────────────────────────
  P3,P7   │   P1     │  P4,P9   │  ... │    P2

  H(K(Pi)) % N  ──→  assigns Pi to slot
  Within slot: FIFO on arrival_time
```

This is structurally similar to a hash table with chaining — each bucket is a micro-queue. Uniform hash distributions minimize bucket imbalance, which translates directly to CPU utilization balance.

---

# 3. Formal Properties — Fairness, Starvation, Determinism

Any new scheduling policy needs to be evaluated against classical criteria. Let's work through them honestly.

## Fairness

A hash scheduler with a *uniformly distributed* hash function distributes processes across time slots with probability close to equal allocation. For *n* processes and *N* slots, expected maximum bucket load is `O(log n / log log n)` — far better than worst-case heap or FIFO under adversarial arrival.

However: fairness is only probabilistic, not guaranteed. Two processes could hash to identical slots for multiple consecutive epochs if the key space is poorly designed. **This is the primary formal weakness.**

## Starvation

With a fixed seed, a process with an unlucky hash value could be perpetually out-ranked. The mitigation: **aging via key mutation** — include a monotonically increasing component (e.g., `wait_ticks`) in the hash key so that a waiting process's hash drifts toward higher priority over time.

```c
// Anti-starvation: age the key by wait duration
K = p.pid ^ (p.arrival_ns >>> 10)
    ^ p.weight
    ^ (MAX_WAIT - p.wait_ticks)   // longer wait → lower hash → higher priority
```

## Determinism and Reproducibility

This is where hash scheduling has a **meaningful advantage** over lottery scheduling. Given the same process state and seed, the scheduling sequence is *perfectly reproducible*. This has concrete value for:

- **Debugging** — deterministic scheduling means deterministic race condition reproduction
- **Security** — PRNG-based lottery schedulers are vulnerable to timing side-channels; hash-based is harder to manipulate
- **Testing** — scheduler behavior can be unit-tested against a fixed set of process states

## Cache Locality

If the hash key includes memory region or page-table base address, processes with overlapping working sets can be co-scheduled — exactly the property that RSS exploits for network flows. This gives hash scheduling a potential edge over purely time-based policies on NUMA hardware.

---

# 4. Hash Scheduling vs. Existing Algorithms

Here's how it compares across the properties that matter most:

**Deterministic?**
FIFO ✅ | Round Robin ✅ | CFS ✅ | Lottery ❌ | Hash ✅

**Starvation-free?**
FIFO ❌ | Round Robin ✅ | CFS ✅ | Lottery ⚠️ Probabilistic | Hash ⚠️ With aging

**O(1) dispatch?**
FIFO ✅ | Round Robin ✅ | CFS ❌ O(log n) | Lottery ❌ O(n) | Hash ⚠️ O(n) naïve / O(1) bucketed

**Priority-aware?**
FIFO ❌ | Round Robin ⚠️ Optional | CFS ✅ | Lottery ✅ | Hash ✅ Via key design

**Cache-locality-aware?**
FIFO ❌ | Round Robin ❌ | CFS ⚠️ Domain hints | Lottery ❌ | Hash ✅ Via key design

**Reproducible trace?**
FIFO ✅ | Round Robin ✅ | CFS ✅ | Lottery ❌ | Hash ✅

**Manipulation-resistant?**
FIFO ❌ | Round Robin ❌ | CFS ❌ Partial | Lottery ⚠️ Partial | Hash ✅ Strong (cryptographic hash)

The standout property: **composability**. Hash scheduling doesn't commit to a single optimization objective — you tune the key, and the policy follows. No existing algorithm offers this degree of structural flexibility at the dispatch layer.

---

# 5. Prototype Hash Scheduler (C sketch)

Here's a more complete prototype — not a real kernel module, but enough to reason about the data structures and the dispatch loop:

```c
/* hash_sched.c — prototype hash-based CPU scheduler sketch */

#define EPOCH_SLOTS   64
#define HASH_SEED     0xdeadbeefcafe1234ULL

typedef struct {
    uint32_t pid;
    uint64_t arrival_ns;
    uint32_t weight;          /* nice-derived priority weight */
    uint64_t wait_ticks;      /* anti-starvation aging counter */
    uint64_t mem_region_hint; /* page-table base for NUMA affinity */
} proc_t;

/* Composite key construction — tune this to tune policy */
static inline uint64_t build_key(const proc_t *p, uint64_t epoch) {
    return   ((uint64_t)p->pid        << 32)
           ^ (p->arrival_ns           >>> 10)
           ^ (p->weight               << 16)
           ^ (MAX_WAIT - p->wait_ticks)        /* aging: longer wait → lower hash */
           ^ (p->mem_region_hint      >>> 12)  /* NUMA hint: page frame bits */
           ^ epoch;                            /* epoch rotation for fairness */
}

/* Bucket-based dispatch: O(n) assignment, O(1) per-slot dispatch */
void hash_assign_slots(proc_t *procs, int n, uint64_t epoch,
                       proc_t *slots[EPOCH_SLOTS]) {
    for (int i = 0; i < n; i++) {
        uint64_t key  = build_key(&procs[i], epoch);
        uint64_t h    = xxhash64(&key, sizeof(key), HASH_SEED);
        int      slot = h % EPOCH_SLOTS;

        /* Chaining: if slot occupied, pick next empty (linear probe) */
        while (slots[slot] != NULL)
            slot = (slot + 1) % EPOCH_SLOTS;

        slots[slot] = &procs[i];
    }
}

/* Dispatch: walk slots in order — this IS the scheduling sequence */
proc_t* hash_next(proc_t *slots[EPOCH_SLOTS], int *cursor) {
    while (*cursor < EPOCH_SLOTS) {
        if (slots[*cursor] != NULL)
            return slots[(*cursor)++];
        (*cursor)++;
    }
    return NULL; /* end of epoch */
}
```

The entire scheduler state is a **flat array of `EPOCH_SLOTS` pointers**. No red-black tree. No heap. No linked list. Recomputed fresh each epoch from process state alone. This has profound implications for verifiability — you can formally specify the complete scheduler state as a pure function of process attributes and epoch number.

---

# 6. Open Problems and Research Vectors

This is where the honest systems researcher has to plant a flag. The idea is compelling, but several hard questions remain unanswered:

## Hash Function Selection

Which hash function? xxHash64 is fast and uniform. SipHash-1-3 provides cryptographic resistance against adversarial key manipulation. MurmurHash3 has excellent avalanche properties. The choice has measurable performance and security tradeoffs at scheduler frequencies (microsecond timescales).

## Key Stability vs. Key Volatility

If the key includes volatile attributes (like `wait_ticks`), the hash changes at every scheduling decision. Does this cause thrashing in cache-affine workloads? What is the optimal "key refresh rate" relative to quantum size?

## Real-Time Guarantees

Hard real-time systems need bounded worst-case latency. A pure hash scheduler offers probabilistic, not deterministic, latency bounds. Can you augment it with a real-time lane — a reserved slot set — while still hashing best-effort tasks?

## Multi-Core Coherence

On SMP systems, each core running a hash scheduler independently may assign the same process to different cores simultaneously. This requires a global epoch counter with lightweight synchronization — a potential scalability bottleneck.

## Security Model

If an adversarial process can observe its own scheduling latency, it may be able to reverse-engineer the hash seed. Using a cryptographic hash (SipHash) with a per-boot secret seed closes this channel — but at a compute cost. This is an **unexplored side-channel model** for scheduler policies.

> The most interesting claim: a hash scheduler with a well-designed composite key is not just a scheduling algorithm — it is a declarative scheduling *policy language*. Change the key composition, change the policy. No code changes required.

This last point deserves emphasis. Modern kernels require code changes, recompilation, or module replacement to change scheduling policy. A hash scheduler could expose **key composition as a runtime-tunable parameter** — giving sysadmins and containerization layers a knob to dial in workload-specific behavior without touching kernel code.

---

# Closing Thoughts

Hash-based scheduling is not a replacement for CFS or a drop-in for real-time kernels. But as a **composable, deterministic, formally verifiable scheduling primitive**, it offers a conceptual lens that existing algorithms don't.

The kernel is one of the few places left where a single elegant idea can touch every process on a machine. Hash scheduling might be one such idea.
---

*Tags: Linux · Operating Systems · Kernel Development · Systems Programming · Computer Science*
