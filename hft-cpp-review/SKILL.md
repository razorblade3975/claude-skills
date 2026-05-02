---
name: hft-cpp-review
description: "Review C++ code for HFT/low-latency anti-patterns on the hot path. Use whenever the user asks to review, audit, lint, profile, or optimize C++ code for latency, throughput, or HFT correctness — even if they don't say 'HFT' explicitly. Trigger on phrases like 'review this for the hot path', 'is this fast enough', 'any perf issues', 'low-latency review', 'trading code review', 'optimize this', or whenever the user shares C++ code in a trading/market-data/order-book/feed-handler/matching-engine context. Flags four categories of issues — (1) poor cache locality, (2) heap allocation on hot paths, (3) STL containers/algorithms that have strictly faster Boost or specialized alternatives, (4) unnecessarily expensive operations on hot paths (string ops, syscalls, virtual dispatch, exceptions, etc.). Produces a structured, prioritized report with concrete suggested fixes."
---

# HFT C++ Hot-Path Review

Review C++ code for low-latency anti-patterns. The output is a structured report with findings grouped by category, each including a severity, a precise file:line reference, an explanation grounded in the actual cost, and a concrete fix.

## When to apply

This skill is for **hot-path code** — the bursty critical sections that run per-tick, per-message, per-order. Cold paths (startup, config loading, logging during shutdown, test fixtures) get a much lighter pass: most rules here either don't apply or apply weakly. Before flagging, ask whether the code is plausibly on the hot path. Heuristics:

- Functions named `on_tick`, `on_message`, `on_book_update`, `process_packet`, `handle_order`, `match()`, anything inside a tight feed-handler loop, anything in a callback registered with a kernel-bypass NIC API.
- Code reachable from a thread pinned to a core, a busy-spin loop, or a `while (true)` polling loop.
- Anything in a `.cpp` whose neighbors include `rdtsc`, `_mm_pause`, `__builtin_prefetch`, `[[gnu::hot]]`, or kernel-bypass headers (`rte_*` for DPDK, `efvi`/`ef_vi` for Solarflare, `ibv_*` for RDMA).

If the code is clearly cold (e.g. parsing a YAML config), say so explicitly and limit the review to correctness issues that happen to overlap (e.g. a config-loaded `std::map` that gets *read* on the hot path is still a problem — the read site is what matters).

## How to do the review

1. **Identify the hot path.** Read the file(s). Note entry points, loops, and any latency-sensitive functions. If the user told you "this function is the hot path", trust them but still glance at callees. If unclear, ask once, briefly. Otherwise assume any function that processes market data, orders, or fills is hot.

2. **Walk through each of the four categories below in order.** For each, scan the code with that lens specifically — don't try to spot all four issues simultaneously, you'll miss things. The categories are ordered roughly by how much latency they typically cost.

3. **For each finding, record:**
   - `severity`: `critical` (definitely costs latency on every call), `high` (costs latency in common cases), `medium` (costs latency under some inputs / cache states), `low` (style / defensive improvement).
   - `category`: one of `cache-locality`, `heap-allocation`, `stl-alternative`, `expensive-op`.
   - `location`: `path/to/file.cpp:LINE` (or a line range).
   - `problem`: 1–2 sentences explaining *why this costs cycles*, not just "this is slow". Reference cache lines, allocator paths, syscalls, branch prediction, etc. concretely.
   - `fix`: a concrete, code-level suggestion. Where reasonable, give a short snippet.

4. **Output the report in the format below.** Don't dump prose — engineers reading this want to scan and act.

5. **Be specific, not preachy.** Don't say "consider using a faster container". Say "`std::unordered_map<OrderId, Order>` here causes a chained-bucket lookup with a pointer chase per probe; replace with `boost::unordered_flat_map` (open addressing, ~2x faster lookup, no per-node allocation)." If you can't justify the cost concretely, drop the finding.

6. **No false positives.** If a `std::string` is constructed *once* at startup and only `.data()` is called on the hot path, that's fine — don't flag it. Latency review credibility dies the moment you cry wolf.

---

## The four categories

### 1. Cache locality

The dominant cost on modern HFT-grade hardware is memory access. An L1 hit is ~4 cycles; a main-memory miss is 200–300+ cycles. Anything that scatters data across cache lines or causes pointer chasing is suspect.

**Flag:**

- **Array-of-structs (AoS) where struct-of-arrays (SoA) would fit the access pattern.** If the hot path reads only `price` and `qty` from a 128-byte `Order` struct in a loop, you're pulling in 64-byte cache lines mostly full of fields you don't read.
- **Pointer-chasing containers in hot loops.** `std::list`, `std::map`, `std::set`, `std::unordered_map` with default config — each node is a separate heap allocation, so traversal is a chain of cache misses. Even a single lookup is 1+ pointer chase per probe in a chained hash map.
- **False sharing.** Two atomics or frequently-written variables on the same 64-byte cache line, accessed by different threads. Look for adjacent `std::atomic<...>` members in a struct without `alignas(64)` / `alignas(std::hardware_destructive_interference_size)` padding.
- **Hot/cold field mixing.** A struct where the first 16 bytes are read every tick but the next 200 bytes (string symbol, debug counters, last-update timestamp) are only touched on rare events. The cold tail evicts useful data from L1.
- **Indirection via `std::unique_ptr` / `std::shared_ptr` / `std::function` in hot loops.** Each deref is a potential miss. `std::function` additionally may heap-allocate for non-SBO captures.
- **Virtual dispatch through a base pointer in a tight loop**, especially when the dynamic type is one of a small known set — the vtable lookup is a load, and the indirect call defeats the BTB if types vary.
- **Iterating over a container of `std::shared_ptr<T>`** where `T` could have been stored inline.
- **Random-access patterns over large data structures** that exceed L2 (~1MB/core typical). Even contiguous storage doesn't help if you stride by more than a cache line and don't prefetch.

**Don't flag:**

- A single struct member read once per call.
- `std::vector<T>` with small `T` — that's the *good* answer most of the time.
- AoS in a function that touches every field of every element (then SoA wouldn't help).

**Fix examples:**

- `alignas(64) std::atomic<uint64_t> counter;` — kill false sharing.
- Split `Order` into `OrderHot { price, qty, side }` and `OrderCold { symbol, client_id, ... }` stored in parallel arrays.
- Replace `std::vector<std::unique_ptr<Tick>>` with `std::vector<Tick>` if Tick is movable and not too large.

### 2. Heap allocation on the hot path

`malloc` / `new` go through tcmalloc/jemalloc/glibc allocator paths that are typically 50–200ns minimum, can take a lock under contention, and fragment over time. On a hot path that targets sub-microsecond latency, **zero allocations** is the goal.

**Flag:**

- **Any `new`, `make_unique`, `make_shared`, `malloc`, `calloc`, `realloc`** inside a function on the hot path.
- **Container operations that allocate**: `vector::push_back` past capacity, `vector::resize` growing, `string` operations producing a new string, `unordered_map::insert` (each insert allocates a node in chained implementations), `std::deque::push_back` (block allocation), any `set`/`map` insert.
- **`std::function` constructed from a lambda with non-trivial captures** — likely heap-allocates for the type-erased target.
- **`std::any` / `std::variant` to types that don't fit SBO** (variant is fine if all alternatives are small; flag if one alternative is large and forces heap on others — actually variant doesn't heap, but `any` does for non-small types).
- **Exception throws** — exception objects allocate on the heap (and unwinding the stack costs hundreds of ns to microseconds even if not caught hot).
- **`std::stringstream`, `std::ostringstream`, `to_string`** — all allocate.
- **`std::shared_ptr` constructed via the two-argument `shared_ptr(new T(...))`** — two allocations (control block + object). `make_shared` is one. Both are still allocations on the hot path, but the two-arg form is strictly worse.
- **`std::sort` on a range that requires temporary buffers** — typically OK, but `std::stable_sort` allocates a temp buffer.
- **Capturing by value in a lambda that escapes**, if the lambda is then wrapped in `std::function`.

**Don't flag:**

- `vector::emplace_back` when the vector was `reserve()`d to a known max upstream — say so explicitly and verify the reserve happened.
- Stack allocations of any size below ~8KB. (Above that, watch out for stack-frame issues, but it's not heap.)
- Allocations in `init()` / constructor / cold paths.

**Fix examples:**

- Pre-`reserve()` vectors at startup with the max expected size.
- Replace `std::unordered_map<K,V>` with a fixed-size open-addressing table or `boost::unordered_flat_map` with `reserve()` upfront.
- Use object pools / free lists for `Order` and `Message` types — recycle, never `delete`.
- Replace error-via-exceptions with `tl::expected<T, ErrorCode>` or a status enum.
- Use `fmt::format_to(buffer, ...)` into a stack `std::array<char, N>` instead of `std::ostringstream`.
- For type erasure, use `function_ref` (non-owning) or a fixed-size SBO-only function type.

### 3. STL alternatives

The C++ standard library trades raw speed for ABI stability, generality, and (for some containers) iterator-invalidation guarantees that nobody actually wants on a hot path. There are strictly faster drop-in (or near-drop-in) replacements.

**Flag and suggest:**

| Stdlib | Replacement | Why |
|---|---|---|
| `std::unordered_map<K,V>` | `boost::unordered_flat_map<K,V>` | Open addressing, no per-node alloc, ~2× faster lookup, far better cache behavior. Iterators invalidate on rehash, which is fine on a hot path. |
| `std::unordered_set<T>` | `boost::unordered_flat_set<T>` | Same reasoning. |
| `std::map<K,V>` | `boost::container::flat_map<K,V>` (sorted vector) | Contiguous storage, binary search; ideal for small-to-medium sizes (<~1k) and read-heavy workloads. |
| `std::set<T>` | `boost::container::flat_set<T>` | Same reasoning. |
| `std::deque<T>` | `boost::circular_buffer<T>` (if bounded) or `std::vector<T>` with index ring | `std::deque` allocates fixed blocks per N elements; `circular_buffer` is one allocation. |
| `std::list<T>`, `std::forward_list<T>` | `std::vector<T>` or intrusive list (`boost::intrusive::list`) | Almost never the right answer on a hot path. Intrusive list avoids the per-node allocation if you genuinely need O(1) splice. |
| `std::shared_mutex` | `boost::shared_mutex` or RW-spinlock for short critical sections | Stdlib `shared_mutex` is heavyweight on most libstdc++ versions. |
| `std::regex` | `re2`, `ctre` (compile-time regex), or hand-written parser | `std::regex` is famously slow and allocates. |
| `std::random_device` / `std::mt19937` per-call | Thread-local PRNG, ideally `xoshiro256**` or `pcg32` | `mt19937` has a 2.5KB state — terrible for cache. |
| `std::chrono::system_clock::now()` on every event | `rdtsc` / `__rdtscp` with a calibrated frequency, or `clock_gettime(CLOCK_MONOTONIC_COARSE)` for non-precision uses | `system_clock::now()` typically calls `clock_gettime(CLOCK_REALTIME)` — a vDSO call, ~20ns, but still strictly more than `rdtsc` (~10 cycles). |
| `std::vector<bool>` | `boost::dynamic_bitset` or `std::vector<uint8_t>` | `vector<bool>` is a proxy mess. |
| `std::queue` over `std::deque` for SPSC/MPMC | `boost::lockfree::spsc_queue` / `boost::lockfree::queue` | Lock-free, cache-aware. |
| `std::hash<K>` for integer-like keys | Identity hash or `boost::hash` with a known-good mixer | `std::hash<int>` is identity on libstdc++/libc++, which is *bad* for power-of-two table sizes — clusters. Boost flat maps handle this internally; if rolling your own table, hash explicitly. |

**Don't flag:**

- `std::vector<T>` — it's almost always correct.
- `std::array<T, N>` — perfect.
- `std::span` — zero overhead.
- `std::optional<T>` for small T — fine, no allocation.

**Note on `absl::flat_hash_map`**: Abseil's flat hash map is also excellent and roughly equivalent to `boost::unordered_flat_map`. If the codebase already uses Abseil, prefer that to avoid mixing dependencies. Mention this when suggesting.

### 4. Unnecessary expensive operations on the hot path

Operations that aren't allocations or cache problems but are still expensive in absolute terms.

**Flag:**

- **String operations on the hot path**: any `std::string` construction, concatenation, comparison of long strings, `substr`, `find`, `to_string`, formatting. If the code stringifies anything per-tick (e.g. for logging), that's a finding — even with allocation aside, the byte-by-byte work isn't free.
- **Logging via `<<` chains or `printf` on the hot path** — formatting is expensive, and most loggers take a lock or hit a global. Flag any logger call inside the hot loop unless it's clearly conditional on a rare error path. Suggest a lock-free async logger (e.g., `quill`, `spdlog` async mode, or a custom binary-format ring buffer that does formatting offline).
- **Syscalls**: `gettimeofday`, `clock_gettime(CLOCK_REALTIME)`, `read`, `write`, `send`, `recv`, `getpid`, anything in `<sys/...>` not behind a vDSO. Even vDSO calls are 15–30ns.
- **`std::mutex::lock()` / `std::lock_guard` on the hot path** when contention is possible. Even uncontended, ~20ns. With contention, microseconds. Suggest single-writer designs, lock-free queues, or seqlocks for read-mostly state.
- **Virtual function calls in a tight loop** when the type set is small and known — switch to `std::variant` + `std::visit`, CRTP, or an enum + switch.
- **Exception-based control flow** — even if not thrown, `try`/`catch` can inhibit some optimizations and increases code size. Throwing is the real killer (hundreds of ns minimum).
- **RTTI (`dynamic_cast`, `typeid`)** — `dynamic_cast` walks the inheritance graph; not constant time across all hierarchies.
- **Division by a non-constant integer** — 20–40 cycles on x86. If the divisor is known at compile time, it'll be strength-reduced; if known at runtime but constant for a while, use libdivide.
- **`std::fmod`, `std::sin`, `std::cos`, `std::log`, `std::exp`** in hot loops — flag and suggest precomputed tables, polynomial approximations, or batched SIMD math (Sleef, xsimd, vectorized libm).
- **Floating-point-to-string and string-to-float** (`std::to_string(double)`, `std::stod`, `strtod`) — extremely slow. Use `std::to_chars` / `std::from_chars` (C++17, allocation-free, ~5–10× faster) or `fast_float` for parsing.
- **Memory allocation hidden inside seemingly-cheap calls** — `std::sregex_iterator`, `std::filesystem::path` operations, `std::locale`-aware functions (anything that touches a locale; `std::tolower(int, locale)`, `std::isdigit` from `<locale>`). Prefer `<cctype>` versions and ASCII-only assumptions where applicable.
- **Reading `errno` after every syscall in a wrapper loop** — not expensive per se, but `errno` is thread-local, often a TLS lookup; if you're checking it 10× per packet, it adds up.
- **Branch-mispredict-prone code** — especially `if (rare_condition)` without `[[unlikely]]` / `__builtin_expect`. This is a soft finding (`low` severity) unless the branch is provably hot and unpredictable.

**Don't flag:**

- Integer arithmetic, bitwise ops, array indexing, comparisons — all 1 cycle.
- `memcpy` / `memmove` / `memset` of small known sizes — the compiler typically inlines them well.
- A single `clock_gettime` per event for timestamping — usually mandatory for HFT correctness; only flag if there are multiple per event or a coarse clock would do.

---

## Output format

Produce the report in this exact structure. If a category has no findings, write "No findings" — don't omit the header.

```markdown
# HFT C++ Review: <file or component>

**Hot path identified:** <brief description, e.g. "`FeedHandler::on_packet` and everything it calls">
**Files reviewed:** <list>

## Summary

- Critical: <count>
- High: <count>
- Medium: <count>
- Low: <count>

## Cache locality

### [SEVERITY] <one-line title>
**Location:** `path/to/file.cpp:LINE`
**Problem:** <1–2 sentences, concrete cost mechanism>
**Fix:**
```cpp
// suggested code or pseudocode
```

<repeat per finding>

## Heap allocation on hot path

<same structure>

## STL alternatives

<same structure>

## Expensive operations

<same structure>

## Notes

<anything that isn't a finding but is worth flagging — e.g. "the `init()` path uses `std::map`; this is fine because it's cold, but if `lookup()` is called per-tick consider migrating">
```

## Worked example

**Input snippet:**

```cpp
class OrderBook {
    std::unordered_map<OrderId, std::shared_ptr<Order>> orders_;
    std::mutex mu_;
public:
    void on_new_order(const std::string& symbol, Order o) {
        std::lock_guard<std::mutex> g(mu_);
        auto p = std::make_shared<Order>(std::move(o));
        orders_[p->id] = p;
        std::cout << "New order " << p->id << " for " << symbol << "\n";
    }
};
```

**Expected output (excerpt):**

```markdown
## Heap allocation on hot path

### [CRITICAL] `make_shared<Order>` per order
**Location:** `order_book.cpp:7`
**Problem:** Every new order allocates an `Order` + control block on the heap. At 100k orders/sec this is 100k allocations/sec on the critical path; each is ~80–150ns and contends with the allocator.
**Fix:** Use an object pool. Pre-allocate a `std::vector<Order>` sized to max expected outstanding orders and store indices in the map.

### [HIGH] `unordered_map::operator[]` insert allocates a node
**Location:** `order_book.cpp:8`
**Problem:** `std::unordered_map` is chained — every insert heap-allocates a node, and lookups pointer-chase through buckets.
**Fix:** Replace with `boost::unordered_flat_map<OrderId, uint32_t>` (storing pool indices), `reserve()`d to the max expected size at startup.

## STL alternatives

### [HIGH] `std::unordered_map` → `boost::unordered_flat_map`
**Location:** `order_book.cpp:2`
**Problem:** See above — chained hashing causes per-node allocation and pointer chasing on every probe.
**Fix:** `boost::unordered_flat_map<OrderId, uint32_t> orders_;` with `orders_.reserve(MAX_ORDERS)` in the constructor.

## Expensive operations

### [CRITICAL] `std::cout <<` on the hot path
**Location:** `order_book.cpp:9`
**Problem:** Synchronous formatted I/O — takes a global lock, formats, and may flush to a TTY/pipe. Easily 1–10μs per call. Also `std::string symbol` parameter passed by reference is fine, but the `<<` of it converts and copies bytes.
**Fix:** Remove from the hot path entirely, or replace with an async lock-free logger (e.g., `quill`) that captures arguments and formats on a background thread.

### [HIGH] `std::lock_guard<std::mutex>` per order
**Location:** `order_book.cpp:6`
**Problem:** Even uncontended, ~20ns; under contention from a separate thread (e.g., a market-data thread also touching the book), microseconds.
**Fix:** If the order book is single-threaded (preferred for HFT), drop the mutex. If multi-threaded, use a single-writer / multiple-reader seqlock or partition the book by symbol so each thread owns a partition.

### [MEDIUM] `const std::string& symbol` parameter
**Location:** `order_book.cpp:5`
**Problem:** Forces callers to either own a `std::string` (allocation upstream) or construct one. The function only logs the symbol — doesn't store it.
**Fix:** `std::string_view symbol`. If the symbol is fixed-width (e.g., 8 bytes), `std::array<char, 8>` by value is even better and fits in a register.
```

## Final reminders

- Stay grounded in concrete cost. "Slow" is not an explanation; "200ns minimum due to allocator path" is.
- It's fine to say a section has no findings. Padding the report with weak findings dilutes the strong ones.
- If the user shares only a snippet without context, ask whether it's truly hot before flagging — but ask once and proceed with a flagged-as-uncertain review rather than blocking.
- If the user's codebase clearly has a house style (e.g., already uses `absl::` everywhere), match it in your suggestions.
