# Could Hashing Replace FIFO and Round Robin?

## A Deterministic, Composable Alternative to Traditional CPU Scheduling

> What if the CPU scheduler wasn't a queue, a tree, or a lottery --- but
> a pure function?

Modern operating systems answer one question thousands of times per
second:

**Which runnable process gets the core next?**

FIFO uses arrival order.\
Round Robin uses time slices.\
CFS uses virtual runtime in a red-black tree.\
Lottery scheduling uses randomness.

But fundamentally, every scheduler implements:

> **f(process_state) → dispatch_order**

This article explores a different framing:

> What if scheduling were implemented as a hash function over composable
> process attributes?

------------------------------------------------------------------------

# Why This Matters

Recruiters and systems engineers care about three things in scheduler
design:

-   Determinism
-   Fairness
-   Performance complexity

Hash-based scheduling introduces something new:

> **Policy composability without structural rewrites.**

Instead of replacing queues or trees, we redefine the scheduling key.

------------------------------------------------------------------------

# The Core Idea

For each runnable process `p`, compute:

    H(K(p))

Where: - `K(p)` is a composite key derived from selected attributes -
`H` is a fast, uniform hash function - The minimum hash wins dispatch

Change the key → change the policy.

No new data structures required.

------------------------------------------------------------------------

# Example Key Designs

**pid only**\
→ deterministic pseudo-random ordering

**pid + arrival_time**\
→ FIFO-like behavior without maintaining a queue

**priority + burst_estimate**\
→ weighted fairness (CFS-style)

**memory_region + pid**\
→ NUMA/cache-locality awareness

The key becomes the policy layer.

------------------------------------------------------------------------

# Two Implementable Variants

## 1. Hash-as-Rank

At each scheduling point: - Compute hash per runnable process - Select
minimum

Simple. Deterministic. Reproducible.

## 2. Hash-Bucketed Epoch Slots

Divide time into N slots:

    slot = H(K(p)) % N

Each slot acts as a micro-queue.

Expected maximum bucket load:

    O(log n / log log n)

This gives probabilistic fairness with bounded imbalance.

------------------------------------------------------------------------

# How It Compares

**Deterministic?**\
FIFO ✅ \| RR ✅ \| CFS ✅ \| Lottery ❌ \| Hash ✅

**Starvation-Free?**\
FIFO ❌ \| RR ✅ \| CFS ✅ \| Lottery ⚠️ \| Hash ⚠️ (aging key)

**O(1) Dispatch?**\
FIFO ✅ \| RR ✅ \| CFS ❌ \| Lottery ❌ \| Hash ⚠️ (O(1) bucketed)

**Reproducible Execution?**\
Lottery ❌ \| Hash ✅

That last property is particularly relevant for: - Debugging race
conditions - Deterministic replay - Side-channel analysis

------------------------------------------------------------------------

# Why Recruiters Should Care

This concept demonstrates:

-   Understanding of kernel scheduling internals
-   Ability to abstract systems into mathematical primitives
-   Awareness of SMP, NUMA, and fairness tradeoffs
-   Security considerations (SipHash vs xxHash, seed secrecy)
-   Practical implementation feasibility (sched_class prototype)

It's not a replacement for CFS.

It's a research-direction primitive.

------------------------------------------------------------------------

# A Tractable Experiment

This is realistically implementable as:

-   A Linux `sched_class`
-   Benchmarked via `hackbench` / `schbench`
-   Compared against `SCHED_OTHER`
-   Evaluated using probabilistic fairness bounds

The scheduler state becomes:

> A pure function of (process_state, epoch)

That is rare in kernel architecture.

------------------------------------------------------------------------

# Closing Thought

Hash scheduling reframes dispatch from:

**"Which queue element is next?"**

to

**"Which process hashes lowest under current policy composition?"**

That shift turns scheduling into a declarative system rather than a
structural one.

And that makes it interesting.

------------------------------------------------------------------------


