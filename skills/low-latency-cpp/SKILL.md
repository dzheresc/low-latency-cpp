---
name: low-latency-cpp
description: Use when designing, writing, reviewing or tuning latency-sensitive C++ (trading, telecom, market data, real-time control): threading, atomics, queues, memory, CPU/OS tuning, networking, and tail-latency measurement.
---

# Low-latency C++ systems engineering

Working guide for code where the time of *each* operation matters, usually
judged at the tail (P99, P99.9, P99.99), not the mean.

Provenance: distilled from Leandro Silveira's six-part Medium series
"Low-Latency C++ Systems Engineering". Section 1 is from Part 1 (which, as
published, ends at its section 4.1); sections 2 to 6 are from Parts 2
to 6 (read in full). Notes marked "Additional practice (general...)" are
not from the articles.

## 0. Ground rules (apply to every change)

1. **Measure first, at the tail.** State the latency budget and percentile
   up front. Optimise only what a profile or a latency histogram points to.
2. **Latency and throughput can conflict.** Batching, pools and extra threads
   usually raise throughput and also raise queueing delay. Pick deliberately.
3. **Variance is the enemy.** A predictable 5 µs beats an average 3 µs with
   500 µs spikes. Remove sources of jitter before shaving mean cost.
4. **Nothing unbounded on the hot path:** no allocation, no syscalls, no
   blocking locks, no exceptions thrown, no logging I/O, no thread creation,
   no page faults, no unbounded loops or containers.
5. **Simplest design that meets the budget.** A single-threaded pipeline
   often beats a multithreaded one. Every advanced technique below is
   applied selectively, where measurement justifies it.

## 1. Architecture and algorithms (from Part 1)

### 1.1 Architect for temporal behaviour, not just function
The goal is not merely "fast" but **efficient and predictable** under the
expected load: 200 µs average with occasional 5 ms outliers can fail a
strict requirement. Alongside the usual design questions (responsibilities,
communication, dependencies, evolvability), ask of every design:
- How many operations does one request need?
- Which components sit on the critical path?
- How many memory accesses per operation?
- Is there synchronisation that isn't needed?
- Can execution time be bounded, or at least characterised statistically?
- What happens to latency as load and contention rise?

Principle: **judge every abstraction, dependency and communication boundary
by its execution-time cost as well as its design benefit.** This is not a
case against modularity; it is architectural discipline applied to cost.

### 1.2 Design the critical path
- Write the path out explicitly (e.g. receive -> decode -> validate ->
  update state -> strategy -> risk -> build order -> send). Every step adds
  latency; the job is to remove unnecessary work from it.
- Trade-off: a chain of separate services (each serialising, hopping the
  network, validating, logging) buys isolation at a large latency cost.
  Consolidating latency-sensitive steps in one process with in-memory
  hand-off cuts that cost but tightens coupling and raises operational
  complexity. Decide from the actual latency requirement.
- Classify components:
  - **Latency-critical** (must finish inside the budget): decoding, state
    updates, decision logic, risk validation, response generation.
  - **Non-critical** (needn't run synchronously): detailed logging,
    historical persistence, analytics, reporting, non-essential monitoring.
- Move non-critical work off the path, e.g. publish compact event records to
  a **preallocated in-memory queue** for async persistence. That forces
  explicit decisions on durability, backpressure, queue capacity and
  failure recovery.

### 1.3 Algorithms: complexity plus physics
- Lower big-O is not automatically faster, but complexity still tells you
  how cost scales with input size. Real time also depends on memory
  locality, branch prediction, data representation and cache behaviour, so
  **pair complexity analysis with benchmarks on the target hardware.**
- Data structure trade-offs: array O(1) indexed access; hash table O(1)
  average lookup but pays for hashing, collisions, indirection, possible
  allocation and less predictable timing; balanced tree O(log n). A sorted
  contiguous array gives cache-friendly O(log n) search; for small, hot
  collections a **linear scan over contiguous memory can beat all of them.**
- **Worst case matters.** Average-case hides risk: a hash table degrades
  badly under adversarial or unlucky collision patterns. Know what
  conditions degrade each structure, especially when input distributions
  are unpredictable.

### 1.4 Abstractions have runtime costs
Templates, generics, RAII, smart pointers, polymorphism, STL containers and
functional style all aid correctness and maintainability; on the hot path,
know what each costs.
- **Virtual dispatch** is an indirect call through the vtable. It is not
  inherently slow and modern CPUs predict it well, but in very hot paths it
  can hurt branch prediction, block inlining and further optimisation, and
  touch more cache. Consider compile-time polymorphism (templates,
  concepts, CRTP, `std::variant`) where the type set is known.

Additional practice (general, not from Part 1):
- Do work ahead of time: precompute tables, pre-size containers, warm
  caches and code paths before the first real event.
- Prefer flat structures: `std::vector`, `std::array`, open-addressing
  hash maps, sorted vectors, over node-based containers.
- Make error handling cheap on the happy path: return codes or
  `std::expected`; keep exceptions for truly exceptional, off-hot-path cases.
- Mark rare branches `[[unlikely]]` and move cold code out of line
  (`[[gnu::cold]]`, `noinline`) so the hot loop stays small.

```cpp
// Static dispatch instead of virtuals on the hot path
template <class Handler>
void on_packet(const Packet& p, Handler& h) {
    switch (p.type) {
        case MsgType::Add:    h.on_add(p.add());    break;
        case MsgType::Cancel: h.on_cancel(p.cxl()); break;
        default: [[unlikely]] h.on_unknown(p);      break;
    }
}
```

## 2. Memory management and data layout (from Part 2)

Fast algorithms and good concurrency still miss the budget if memory access
is poor. Allocation, cache misses, pointer indirection, fragmentation and
synchronisation on shared data are all latency costs. Central principle:
**organise memory around how the system actually processes data**, and
design allocation around the system's execution needs instead of leaving
it to each operation.

### 2.1 Allocation is an architectural decision
`new`/`delete` on the hot path has variable cost (allocator state, thread
contention, OS behaviour). Allocate before processing starts and reuse.

```cpp
// Bad: allocation per event
void process_event(const Event& e) {
    auto* buf = new char[4096]; decode(e, buf); process(buf); delete[] buf;
}
// Better: owned, reused buffer (now decide ownership, reentrancy, threads)
class EventProcessor {
    std::array<char, 4096> buf_{};
public:
    void process_event(const Event& e) { decode(e, buf_); process(buf_); }
};
```

### 2.2 Preallocation and pools
- If the maximum batch size is known, `reserve()` it (it reserves capacity
  but constructs nothing; lifetimes are still yours). For fixed sizes use
  `std::array`.
- Pools fit when object sizes are predictable, allocation is repetitive,
  the number of live objects is bounded, and allocation overhead and
  fragmentation must be controlled. Variants: fixed-size block pools, slab
  allocators, free lists, region/arena allocators, thread-local arenas.
  Compare them on allocation cost, footprint, fragmentation and concurrency.
- A real pool must handle construction/destruction, object identity
  (hand out stable addresses; don't move objects back into storage) and
  concurrent access. The sketch below keeps identity via a free list:

```cpp
template <class T, std::size_t N>
class Pool {                       // fixed-size, no allocation after ctor
    std::array<T, N> slots_;
    std::array<T*, N> free_;
    std::size_t top_ = N;
public:
    Pool() { for (std::size_t i = 0; i < N; ++i) free_[i] = &slots_[i]; }
    T*   acquire() noexcept { return top_ ? free_[--top_] : nullptr; }
    void release(T* p) noexcept { free_[top_++] = p; }
};   // single-threaded; T default-constructed up front
```

### 2.3 Cache locality and data-oriented design
- Cost depends on where data lives (L1 < L2 < L3 << RAM), so design for
  how data is laid out, not just how much is processed.
- **Contiguous over pointer-based:** `std::vector<Order>` beats
  `std::vector<Order*>` for iteration (spatial locality, no indirection).
  Keep pointers only when you need independent lifetimes, stable addresses
  or polymorphism. Decide from access pattern and ownership model.
- **AoS vs SoA:** Array-of-Structures suits operations that touch most
  fields of one object; Structure-of-Arrays suits scanning one field across
  many objects, and vectorises well (e.g. summing traded quantity reads only
  `quantities`). Pick by dominant access pattern, not OO modelling taste.

```cpp
struct OrderBook {                 // SoA
    std::vector<double> prices;
    std::vector<int>    quantities;
    std::vector<int>    instrument_ids;
};
```

### 2.4 Fragmentation and lifetime
- External fragmentation (free space split into small blocks) and internal
  (blocks larger than needed) both grow in long-running processes with
  churn. Counter: fewer distinct allocation sizes, controlled lifetimes.
- **RAII gives deterministic lifetime, not deterministic time.** A
  destructor may free memory, take locks or do I/O. Know what runs at
  construction and destruction on the hot path (defer heavy teardown).
- Avoid copies: pass large read-only objects by `const&` (mind the lifetime
  of the referent), use moves for ownership transfer. Make ownership and
  data movement explicit.

### 2.5 Alignment and false sharing
- Cache lines are commonly 64 bytes (hardware-dependent; see
  `std::hardware_destructive_interference_size`).
- False sharing: independent variables written by different cores on the
  same line cause coherence traffic. Give each writer its own line, then
  verify the gain on the target CPU and workload.

```cpp
struct alignas(64) Counter { std::atomic<std::uint64_t> value{0}; };
struct Counters { Counter thread_a; Counter thread_b; };   // separate lines
```

### 2.6 Lock-free memory reclamation
Lock-free structures swap blocking for a reclamation problem: one thread
may free a node another still points to. Options, each with different cost
and guarantees: hazard pointers (readers announce what they're using),
epoch-based reclamation (free only after all threads pass a safe epoch),
RCU, reference counting where appropriate. **Lock-free doesn't remove
memory-management complexity; it changes its nature.**

### 2.7 Predictable memory access
Variance sources: cache misses, page faults, TLB misses, allocation,
reclamation, NUMA remote access, coherence traffic. Identify which can
happen on the critical path. Mitigations: allocate everything at init,
`mlockall()` (needs rlimits/OS config), pre-touch pages to avoid first-touch
faults, NUMA-local placement. These reduce variability; they don't fix it.

### 2.8 Measuring memory behaviour
Measure, don't assume. Tools: `perf` (HW counters, cache events), Intel
VTune, Valgrind/Cachegrind, Google Benchmark, AddressSanitizer (memory
errors), Heaptrack (allocation profiling). Metrics: cache miss rate,
allocation frequency and cost, bandwidth, TLB misses, page faults,
footprint, latency distribution under memory pressure. Benchmark with
realistic data: a structure that shines on a small synthetic set can
collapse once the working set exceeds cache.

Additional practice (general, not from Part 2): huge pages for large
tables to cut TLB misses; `std::pmr::monotonic_buffer_resource` as a
standard-library arena; NUMA first-touch from the pinned owning thread.

## 3. Hardware and CPU (from Part 3)

Two implementations with the same big-O can differ widely because of cache
use, branch prediction, instruction scheduling, access patterns and CPU
topology. Hardware knowledge shapes the design from the start, not just
late tuning. Central principle: **organise computation so the processor
does useful work with minimal waiting and predictable resource access**;
fewer instructions is not the goal in itself.

### 3.1 Pipelines, ILP and out-of-order execution
CPUs pipeline, issue several instructions per cycle, execute out of order
and speculate, but only if the code exposes independent work. Long
dependency chains, cache misses and resource contention stall them.
Count dependencies, not just instructions.

```cpp
int chain(int a, int b)        { int x = a + b; int y = x * 2; return y + 10; } // serial
int split(int a, int b, int c, int d) { int x = a + b; int y = c * d; return x + y; } // x, y in parallel
```
(Same idea for reductions: several independent accumulators break the
loop-carried chain.)

### 3.2 Caches
- L1 (smallest, fastest) < L2 < L3 (often shared) << RAM. Exact sizes and
  costs are CPU-specific; look them up for the target.
- Exploit temporal locality (reuse soon) and spatial locality (use
  neighbours). Sequential walks over `std::vector` suit caches and the
  hardware prefetcher; `std::list` traversal is pointer chasing.
- **A cache miss often costs more than the arithmetic you are trying to
  save.** Fewer operations with more misses can be slower. Look at
  working-set size, access pattern, layout, cache-line utilisation,
  indirection and reuse before blaming computation.

### 3.3 Branch prediction
- Mispredictions throw away speculative work. Cost depends on the **input
  distribution**: the same loop is fast on mostly-positive data and slow on
  random data, so benchmark with realistic distributions.
- Branchless forms can help, but the compiler may not emit them, and they
  can add instructions or dependencies. Check the assembly and measure.

```cpp
int count_positive(const std::vector<int>& v) {
    int n = 0;
    for (int x : v) n += (x > 0);      // no explicit branch in source
    return n;
}
```

### 3.4 SIMD
- One instruction over many elements: pricing, risk, signal processing,
  numerics. AVX2 = 256-bit registers = 8 floats per op; AVX-512 doubles that
  where available.
- Simple contiguous loops may auto-vectorise; intrinsics give control but
  need a main loop plus a scalar or masked tail.
- **SIMD is a data-layout decision**: SoA lets you load consecutive values
  of one field. Also weigh vector width, alignment, AVX2 vs AVX-512
  availability, FP precision, setup overhead and portability.

```cpp
#include <immintrin.h>
void add(const float* a, const float* b, float* r, std::size_t n) {
    std::size_t i = 0;
    for (; i + 8 <= n; i += 8)
        _mm256_storeu_ps(r + i, _mm256_add_ps(_mm256_loadu_ps(a + i),
                                              _mm256_loadu_ps(b + i)));
    for (; i < n; ++i) r[i] = a[i] + b[i];       // scalar tail
}
```

### 3.5 NUMA
On multi-socket (and some large single-socket) machines, remote memory
costs extra latency and interconnect bandwidth. Plan CPU placement,
allocation policy, thread affinity, first-touch allocation, data ownership
and cross-node traffic. Inspect with `numactl --hardware`. Matters for
large working sets and multi-socket boxes; often irrelevant on small ones.

### 3.6 TLB and huge pages
TLB misses force page-table walks; they bite with large, scattered working
sets. Check working-set size, locality, page size and allocation patterns.
Huge pages (2 MiB / 1 GiB vs 4 KiB) cut translations but bring allocation,
fragmentation, reservation and ops complexity: **adopt them on measured
TLB pressure, not by default.**

### 3.7 Affinity and topology
Pinning (see 4.8) helps when threads have stable roles, cache locality
matters, migration is a measured problem and topology is understood.
Pinning everything indiscriminately removes scheduler flexibility and can
be slower. Account for physical vs logical CPUs, SMT siblings, NUMA
boundaries, interrupt handling, other processes, and the process's allowed
CPU set.

### 3.8 Compilation
- Optimisations: inlining, unrolling, constant propagation, dead-code
  elimination, auto-vectorisation, scheduling, LTO.
- `-O2` broad; `-O3` adds aggressive loop transforms; **`-Ofast` can break
  standards and IEEE float semantics, so be wary in finance and science.**
- `-march=native` uses the build machine's ISA; binaries may not run on
  older CPUs. Have an explicit target-architecture strategy for deployment.

### 3.9 Measuring hardware behaviour
Tools: `perf` (HW counters), Intel VTune, AMD uProf, Compiler Explorer
(assembly), Google Benchmark, `numactl`.
`perf stat -e cycles,instructions,cache-references,cache-misses ./app`
plus CPU-specific events for branch misses and TLB. Counters have overhead
and platform-specific meaning: repeat under controlled conditions.

Guidelines: know the target microarchitecture; shorten dependency chains;
design for cache locality; check misses before arithmetic; test branches
with real distributions; SIMD where layout justifies it; respect NUMA;
watch TLB on big data; pin only on evidence; read the assembly of hot code;
benchmark compiler flags; validate on the deployment hardware. Optimise
only where evidence shows a meaningful benefit.

Additional practice (general, not from Part 3): frequency scaling and deep
C-states add wake-up jitter; on latency-critical hosts use the
`performance` governor and limit C-states. Consider PGO.

## 4. Concurrency and operating system (from Part 4)

Central principle: **low-latency concurrency is not about maximising thread
count; it is about coordinating execution with minimal contention,
scheduling uncertainty and unnecessary communication.** Make resource
ownership explicit and align execution with what the OS and CPU can do.

### 4.1 Concurrency is not parallelism
- Concurrency = overlapping tasks (possible on one core). Parallelism =
  truly simultaneous execution on several cores.
- Map the pipeline (decode -> state update -> signal -> risk -> order ->
  metrics) and mark which steps are independent and which need strict order.
- Parallelism costs: synchronisation, data sharing, cache-coherence traffic,
  context switches, work distribution, inter-thread messaging. **More
  threads do not automatically lower latency.**

### 4.2 Thread architecture
- Never create/destroy threads per event. Use long-lived workers.
- General thread pools add queueing delay and sync overhead. For strict
  latency prefer: dedicated workers, fixed thread-to-core mapping, bounded
  queues, explicit ownership of each task/datum, specialised pipelines.

### 4.3 Locks and contention
- Contention turns into variance. Keep critical sections as small as
  correctness allows: copy or swap under the lock, compute outside it.
- `std::shared_mutex` helps read-heavy state but is not automatically
  faster; it depends on contention and fairness. Measure.

```cpp
Snapshot snap;
{ std::lock_guard lk(m); snap = shared_state; }   // tiny critical section
auto result = compute(snap);                       // heavy work unlocked
```

### 4.4 Atomics and the memory model
- `relaxed`: atomicity only (counters, stats). No ordering of other data.
- `acquire` load / `release` store: the standard publish/consume pair.
- `acq_rel` for read-modify-write that both consumes and publishes.
- `seq_cst`: default and safest; use when you can't prove a weaker order.
- `consume` is treated as `acquire` by compilers; don't rely on it.
- Weaker orders are a correctness risk. Each non-`seq_cst` order in review
  needs a one-line comment saying what it pairs with.

```cpp
int data;                       // plain payload
std::atomic<bool> ready{false};
// producer
data = 42;
ready.store(true, std::memory_order_release);
// consumer
while (!ready.load(std::memory_order_acquire)) { /* spin / pause */ }
use(data);                      // guaranteed to see 42
```

### 4.5 Lock-free and wait-free
- Built from atomics and CAS (`compare_exchange_weak` in loops,
  `_strong` for single attempts).
- **Lock-free:** some thread always progresses; one thread may retry forever.
  **Wait-free:** every op finishes in a bounded number of its own steps
  (stronger, but much harder to build).
  Neither bounds wall-clock time (scheduling, interrupts, cache misses).
- **ABA:** value goes A->B->A and a CAS wrongly succeeds. Mitigate with tagged
  pointers / version counters, hazard pointers, or epoch-based reclamation.
- **Memory reclamation is part of the design**, not an afterthought. A push
  is easy; a safe pop is not.

```cpp
void push(std::atomic<Node*>& head, Node* n) {
    n->next = head.load(std::memory_order_relaxed);
    while (!head.compare_exchange_weak(n->next, n,
            std::memory_order_release, std::memory_order_relaxed)) {}
}   // publication only: pop needs ABA + reclamation handling
```

### 4.6 Producer/consumer: the SPSC ring buffer
The workhorse pattern (e.g. network thread -> processing thread). Benefits:
preallocated, bounded, no per-event allocation, sequential access, simple
sync. Get right: capacity (power of two), full/empty detection, acquire/
release indices, indices on separate cache lines, a backpressure policy.

```cpp
template <class T, std::size_t N>            // N must be a power of two
class Spsc {
    static_assert((N & (N - 1)) == 0);
    alignas(64) std::atomic<std::size_t> head_{0};   // written by consumer
    alignas(64) std::atomic<std::size_t> tail_{0};   // written by producer
    alignas(64) std::array<T, N> buf_;
public:
    bool push(const T& v) noexcept {
        auto t = tail_.load(std::memory_order_relaxed);
        if (t - head_.load(std::memory_order_acquire) == N) return false; // full
        buf_[t & (N - 1)] = v;
        tail_.store(t + 1, std::memory_order_release);
        return true;
    }
    bool pop(T& out) noexcept {
        auto h = head_.load(std::memory_order_relaxed);
        if (h == tail_.load(std::memory_order_acquire)) return false;     // empty
        out = buf_[h & (N - 1)];
        head_.store(h + 1, std::memory_order_release);
        return true;
    }
};
```
(Further step: cache the other side's index locally to avoid reading the
shared atomic on every call.)

**Backpressure** when full: block the producer, drop (newest or oldest),
flow-control upstream, reject work, or add capacity. Choose from the data's
semantics: stale market data can be dropped; orders and control commands
usually cannot.

### 4.7 OS scheduling
Jitter sources: preemption, context switches, migration, interrupts, kernel
work, noisy neighbours, priority inversion.
- `SCHED_OTHER` (fair), `SCHED_FIFO` / `SCHED_RR` (real-time priorities),
  `SCHED_DEADLINE` (runtime/period budget).
- A real-time thread that never yields can starve everything else on that
  core, including kernel housekeeping. `SCHED_DEADLINE` does not make the
  whole application hard real-time.

### 4.8 Affinity and isolation
```cpp
bool pin_to_cpu(int cpu) {
    cpu_set_t set; CPU_ZERO(&set); CPU_SET(cpu, &set);
    return pthread_setaffinity_np(pthread_self(), sizeof set, &set) == 0;
}   // validate cpu id and check the return value in production
```
- Pinning cuts migration and keeps caches warm.
- Isolation toolkit: cpusets, `isolcpus`/`nohz_full`/`rcu_nocbs`, IRQ affinity
  (steer NIC and other IRQs to housekeeping cores), keep housekeeping cores.
- Isolation is not determinism: SMIs, NMIs, firmware and shared hardware
  (LLC, memory bandwidth, hyper-thread siblings) still interfere.

### 4.9 Real-time Linux
- Stock Linux gives no hard deadlines. PREEMPT_RT makes most of the kernel
  preemptible and threads interrupt handlers, so priorities are honoured
  more reliably.
- Be precise in requirements: **low latency** (minimise response time) vs
  **soft real-time** (occasional misses degrade) vs **hard real-time** (a miss
  is failure). P99.99 < 1 ms trading is not hard real-time; a safety
  controller may be.
- Choose the architecture and OS from the actual failure model.

### 4.10 Page faults
```cpp
if (mlockall(MCL_CURRENT | MCL_FUTURE) != 0) { /* check RLIMIT_MEMLOCK */ }
// then pre-touch: write every page of pools, rings, stacks before go-live
```
Locking stops paging but not cache/TLB/NUMA latency.

### 4.11 OS-level observability
Tools: `perf` (incl. `perf sched`), `top/htop`, `pidstat -t`, `ps -eLo
pid,tid,cls,rtprio,psr`, `taskset`, `numactl`, `cyclictest` (wake-up latency
of the box), `ftrace`, eBPF (`bpftrace`, `runqlat`, `offcputime`).
Track: context switches and migrations, scheduling delay, lock and atomic
contention, queue depth and queueing time, page faults, interrupts, per-
thread CPU. Goal: explain *why* a thread was late, not only how late.

## 5. Networking and communication (from Part 5)

Network communication is part of the critical path, not just
infrastructure. Central principle: **low-latency networking is not about
using the fastest technology; it is about minimising the time from when
information is produced to when it is usefully consumed.** Understand the
whole path, find the dominant cost, and pick the simplest architecture
that meets the target.

### 5.1 Decompose the path, measure it end to end
Path: app builds message -> serialise -> syscall -> kernel stack -> NIC ->
wire -> NIC -> kernel -> wake/schedule app -> decode -> process.
`T_total = T_app + T_serialise + T_kernel + T_network + T_receive + T_process`.
**Measure the whole path before optimising a part**: faster serialisation
is worthless if the stack or scheduling dominates. Keep network latency
and application latency apart; packets can arrive fast and still wait.

### 5.2 TCP vs UDP
- **TCP:** reliable, ordered byte stream, retransmission, flow and
  congestion control, handled by the OS. Costs: retransmit and congestion
  delays, and **head-of-line blocking** (one late segment holds back later
  data). Use when reliable ordering matters more than the last µs:
  transactions, control/config, APIs, backend links.
- **UDP:** datagrams with message boundaries; no delivery, ordering, dedup
  or retransmit guarantees. Use for market data, telemetry, voice/video,
  packet processing, anywhere fresh data beats stale (take the newest
  price rather than wait for a lost older one). UDP is not automatically
  low latency: loss, congestion, buffering and app processing still delay.
  Reliability is yours to build: sequence numbers, gap detection,
  retransmit requests, periodic snapshots.
- Choose the transport that minimises the **total** cost of meeting the
  requirements, not the one with fewer features.

### 5.3 Binary protocols and serialisation
- Text formats (JSON, XML) aid debugging and interop but cost parsing,
  conversions and allocation. Binary options: Protocol Buffers,
  FlatBuffers, MessagePack, custom layouts.
- **Never send a raw C++ struct as the wire format.** Define explicitly:
  field order, integer widths, byte order, alignment/padding, float
  representation, versioning, validation, cross-architecture compatibility.
  Encode and decode through explicit functions with a fixed byte layout.
- Goal: minimise total time turning state into bytes and back (size,
  parsing, allocation, CPU, and serialisation variability), not only size.

```cpp
struct MarketData { std::uint64_t ts; std::uint32_t instrument; double bid, ask; };
// Fixed little-endian layout: 8 + 4 + 8 + 8 = 28 bytes, no padding, no allocation.
inline void put_le(std::byte*& p, std::uint64_t v, int n) {
    for (int i = 0; i < n; ++i) *p++ = std::byte(v >> (8 * i));
}
std::size_t encode(const MarketData& m, std::span<std::byte, 28> out) {
    auto* p = out.data();
    put_le(p, m.ts, 8); put_le(p, m.instrument, 4);
    put_le(p, std::bit_cast<std::uint64_t>(m.bid), 8);   // assumes IEEE-754
    put_le(p, std::bit_cast<std::uint64_t>(m.ask), 8);
    return 28;
}
```
(Decode mirrors this and validates length and version before reading.)

### 5.4 Zero-copy
- Each copy costs CPU, memory bandwidth, cache pollution, maybe an
  allocation, and adds jitter; at millions of small messages per second the
  system can spend more time moving data than processing it.
- Linux options, not interchangeable: `sendfile()` (fd to fd, no user
  buffer), `splice()` (fd to fd via kernel buffers), `MSG_ZEROCOPY` (send
  without copying payload into the kernel; you must handle completion
  notifications before reusing buffers), `io_uring` (async I/O, fewer
  syscalls).
- Zero-copy is not zero overhead: buffer ownership, synchronisation and
  completion handling remain. Adopt only if the saved copies outweigh the
  complexity.

### 5.5 Kernel bypass and DPDK
- Bypass cuts syscalls, kernel transitions, stack processing, copies and
  scheduling jitter, but the app or framework now owns packet buffers,
  queue assignment, polling, affinity, NIC configuration, TX/RX and
  resource isolation. **It's an architectural trade-off, not a default**;
  ordinary services are usually better served by sockets.
- DPDK: user-space packet processing with poll mode drivers, huge pages,
  mempools, lock-free rings, burst processing, dedicated cores.
- **Polling vs interrupts:** a dedicated thread spinning on the RX queue
  removes interrupt overhead and jitter but burns a core even when idle.

```cpp
while (running) {                                   // conceptual PMD loop
    auto n = receive_burst(rx_queue, bufs, MAX_BURST);
    for (std::size_t i = 0; i < n; ++i) process_packet(bufs[i]);
}
```
- Packet buffers come from preallocated pools, never per-packet malloc.
- Fits NFV, telecom, software routers/switches, security appliances,
  high-rate packet processing, specialised market-data gateways. Justify it
  by measured need; count the CPU, hardware compatibility and maintenance.

### 5.6 RDMA
- Moves data between **registered memory regions** via the NIC with
  little remote-CPU involvement. Flow: register memory, set up resources
  (queue pairs), post a work request, NIC transfers, completion queue
  notifies.
- Operations: send/receive, RDMA write, RDMA read, remote atomics.
  Transports: InfiniBand, RoCE, iWARP, each with its own infrastructure.
- Your responsibilities: registration and lifetime, queue pairs,
  completion-queue processing, synchronisation, errors, flow control,
  fabric configuration. **It's a different communication architecture,
  not a faster TCP socket.**

### 5.7 NIC features
- **RSS** spreads flows across RX queues and cores; still handle
  connection affinity, ordering, shared state and sync cost.
- **Hardware timestamps** separate NIC time from scheduling and software
  delay (arrival, transmit, RTT, distributions); mind clock sync and
  timestamp reference domains.
- **Interrupt moderation** (coalescing) cuts interrupt load but delays
  notification; review it for latency-critical traffic (`ethtool -C`).
- **Queue topology:** map NIC queues, cores, NUMA nodes, buffers and
  threads together; a bad mapping wastes good hardware.

### 5.8 Measuring
- Not just average RTT: P50/P95/P99/P99.9/max, throughput, loss, jitter,
  CPU use, queue depth, under congestion, loss and high CPU load.
- Tools: iperf3, ping, tcpdump, Wireshark, ethtool, ss, perf, eBPF tools,
  DPDK test apps, RDMA perftest tools.
- Method: baseline with plain sockets; measure serialise/deserialise
  separately; measure app latency and network RTT separately; vary packet
  size and message rate; add realistic concurrent load; track loss and
  queues; compare percentiles; repeat under controlled conditions.
- **Batching raises throughput but adds per-packet wait**; decide from
  the latency budget and traffic shape.

### 5.9 Architecture guidelines
Pipeline: NIC -> receive -> decode -> validate -> logic -> state ->
encode -> transmit -> NIC, each stage with explicit responsibility.
1. Minimise the critical path: no needless transforms, copies or sync
   between receive and processing.
2. Predictable binary formats with explicit layout, validation, versioning.
3. Separate data and control planes: config, logging, monitoring and admin
   must not disturb the data path.
4. Manage memory deliberately: preallocated buffers and pools.
5. Preserve ordering where the application needs it, even when parallel.
6. Define backpressure: bounded queues, controlled drops, flow control,
   load shedding.
7. Optimise from evidence: don't adopt DPDK, RDMA or bypass for prestige.
8. Design for loss, outages, congestion and hardware failure up front.

Additional practice (general, not from Part 5): on kernel sockets consider
`TCP_NODELAY`, socket buffer sizing and `SO_BUSY_POLL`; other bypass
stacks include Solarflare/Onload/ef_vi and AF_XDP.

## 6. Performance engineering and observability (from Part 6)

Central principle: **low-latency engineering is empirical. A system is low
latency because its behaviour has been measured, understood and validated
against explicit requirements, not because its design looks efficient.**
The goal is performance you can explain, measure, reproduce and trust.

### 6.1 Budgets from day one
Performance engineering is not a final tuning phase. Decompose the
end-to-end requirement into per-stage budgets (illustrative: receive 100,
decode 50, validate 50, state 100, strategy 200, risk 150, order + send 150,
safety margin 200 = 1000 µs). One stage at twice its budget can sink the
whole path. Budget more than time: memory, CPU, allocation rate, bandwidth,
queue depth, throughput, error/timeout rate. Turn "make it faster" into
"P99 of X stays below T under workload W".

### 6.2 Think in distributions
- Averages hide tails. Two systems with P50 80 vs 90 µs can have P99.9 150
  vs 1200 µs; the second is worse for latency-critical work.
- Report P50/P95/P99/P99.9/P99.99 and max. P99.9 = 2 ms means about 1 in
  1000 operations takes at least 2 ms.
- Tail causes: scheduling, page faults, cache misses, allocation, lock
  contention, interrupts, retransmits, queue build-up, background
  maintenance, NUMA. Ask why it is sometimes slow, not only how fast it
  usually is.
- Statistics: P99.99 needs a large number of samples; short runs can't
  characterise rare events. Max grows with run length and is not a stable
  guarantee.

### 6.3 Three levels of benchmark
- **Micro** (lookup, allocation, serialisation, atomics, queue ops): Google
  Benchmark; use `benchmark::DoNotOptimize` to keep results alive. Caches,
  memory pressure, contention and inputs differ from production.
- **Component** (decoder, event queue, pool, packet pipeline): more
  realistic.
- **End to end** (input event -> output event): the one that counts. A
  component win may not show up end to end; it can just move the
  bottleneck.

```cpp
static void BM_Sum(benchmark::State& st) {
    std::vector<int> v(10'000, 42);
    for (auto _ : st) {
        int s = 0; for (int x : v) s += x;
        benchmark::DoNotOptimize(s);
    }
}
BENCHMARK(BM_Sum);
```

### 6.4 Profiling: find the real bottleneck
- Benchmarks say how fast; profilers say where time goes.
- CPU: `perf record -g ./app && perf report`; `perf stat` for cycles,
  instructions, IPC, branches/misses, cache refs/misses (instruction
  bound vs branch bound vs memory bound).
- Flame graphs show aggregated stacks: wide = expensive path; good for
  surprises, library overhead, comparing profiles. They are not per-request
  latency.
- Memory: Heaptrack, Valgrind Massif, ASan, LSan, custom allocation
  counters. Allocation profiling matters most, since allocation adds
  unpredictable cost.

### 6.5 Hardware counters
Cycles, instructions, IPC (instructions retired / cycles), cache, branch
and TLB misses, context switches, migrations. Low IPC suggests waiting on
dependencies or memory, but IPC is not app performance: fewer, slower
instructions can still lose. A "simple" function stalled on memory points
to layout fixes (contiguous storage, less indirection, hot/cold field
split, less data movement). NUMA: `numactl`, `numastat`, `perf`.

### 6.6 Observability without hurting the hot path
- Metrics: request/message latency, P99/P99.9, throughput, queue depth,
  packet loss, CPU, memory, allocation rate, errors (Prometheus + Grafana).
  Don't ship every event to a central system from the hot path; separate
  critical-path measurement, background aggregation, operational
  monitoring and detailed diagnostics.
- Logs: never synchronous I/O on the critical path. Hot thread -> bounded
  log queue -> background writer -> storage. Define the full-queue policy
  (drop low-priority diagnostics, backpressure, reserve room for critical
  events). Async logging lowers interference but isn't free and doesn't
  guarantee persistence.
- Traces (OpenTelemetry) show where time goes across services; sample,
  enable selectively or use low-overhead instrumentation. **Observability
  must not become a latency source itself.**

### 6.7 Measuring latency correctly
- Use `std::chrono::steady_clock` (monotonic) for elapsed time, never wall
  clock. For very short operations timer overhead dominates: use a
  benchmark framework or repeat.
- Place timestamps at defined events: `T_e2e = T_completion - T_arrival`,
  not just around the processing function. Across machines mind clock sync
  and clock domains; hardware timestamps need careful source handling.
- **Coordinated omission:** a closed-loop load generator (next request after
  the last completes) slows down with the system and hides stalls. Use an
  open-loop generator (requests on a schedule) when production traffic
  arrives independently, and measure from the intended send time.

```cpp
using clk = std::chrono::steady_clock;
const auto t0 = clk::now();                 // at a defined arrival event
handle(msg);
hist.record(std::chrono::duration_cast<std::chrono::nanoseconds>(
                clk::now() - t0).count());  // preallocated histogram; report off-path
```

### 6.8 Stress, saturation and soak
- Stress: high rates, bursts, large and tiny messages, many clients, CPU
  saturation, memory pressure, congestion, loss, thread contention,
  background jobs.
- Batching trades throughput for queueing delay; weigh batch size,
  efficiency, queueing, memory and tail latency.
- **Queueing delay climbs steeply near capacity** while throughput looks
  flat: test across a range of utilisation, not just max throughput.
- Soak tests (long runs) expose leaks, resource exhaustion, slow
  fragmentation, queue creep, thermal throttling, maintenance jobs and
  gradual degradation. Sustained matters as much as peak.

### 6.9 Production validation and regression control
- Production adds traffic mixes, shared infrastructure, hardware variance,
  congestion, OS updates, config changes. Validate for the whole lifecycle.
- Regressions: compare before/after (e.g. P99 100 -> 180 µs, +20% memory,
  -15% throughput, more allocations, more cycles per message), allowing
  for noise, workload and hardware differences, and statistical
  uncertainty.
- CI pipeline: production-like build flags, tests, micro, component and
  end-to-end benchmarks, baseline comparison, regression analysis,
  published reports. **Use dedicated benchmark machines; shared CI runners
  distort results.**
- Dashboards: latency P50-P99.9, msgs/s, loss, retransmits, queue depth,
  CPU, cycles, context switches, RSS, allocation rate, page faults, lock
  contention, errors, timeouts, drops. **Alert on sustained P99 above the
  SLO**, not on blips in the average.

### 6.10 Workflow
1. Define requirements: end-to-end and tail latency, throughput, memory,
   CPU, reliability, operating conditions.
2. Baseline with representative load; record hardware, OS, compiler and
   flags, app config, workload and method.
3. Find bottlenecks with profiles and counters (CPU, misses, branches,
   allocation, sync, scheduling, network).
4. State a hypothesis ("contiguous storage will cut P99 via locality"),
   not "this code looks complicated".
5. Change one significant thing at a time; rerun under the same conditions;
   compare the whole distribution.
6. Validate under sustained load and failures; keep correctness,
   reliability and maintainability.

Toolkit: Google Benchmark; perf, VTune, uProf; FlameGraph; Heaptrack,
Valgrind; ASan/UBSan/TSan; Prometheus, Grafana; OpenTelemetry; Wireshark,
tcpdump, iperf3; eBPF, ftrace, pidstat; Compiler Explorer. No single tool
gives the full picture. **Sanitizers and heavy profilers change behaviour:
never treat their numbers as production latency.**

Additional practice (general, not from Part 6): for nanosecond timing on
x86, `rdtsc`/`rdtscp` with a calibrated invariant TSC; HdrHistogram for
recording; `perf c2c` to find false sharing.

## Review checklist

- [ ] Latency budget and target percentile stated; change backed by a measurement.
- [ ] No allocation, syscall, blocking lock, throw, I/O or thread creation on the hot path.
- [ ] Containers flat and preallocated; hot structs cache-line aware; shared writers padded apart.
- [ ] Every critical section minimal; no lock held across I/O or heavy compute.
- [ ] Every non-`seq_cst` atomic has a stated pairing; lock-free code handles ABA and reclamation.
- [ ] Queues bounded, with an explicit and justified full-queue policy.
- [ ] Threads long-lived, owned, pinned where measured to help; RT priorities can't starve the core.
- [ ] Memory locked and pre-touched; NUMA placement correct.
- [ ] Wire format explicit (widths, byte order, padding, version, validation); no raw structs on the wire; data and control planes separated.
- [ ] Results reported as a latency histogram under realistic load, with OS metrics explaining outliers.
- [ ] Benchmarks open-loop where traffic is, with enough samples for the percentile claimed; run on dedicated hardware; baseline recorded; one change at a time.
- [ ] Logging, metrics and tracing kept off the critical path (bounded queues, sampling).
