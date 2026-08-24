# mini-orderbook

A small **limit order book with price-time priority matching**, written in
modern C++ (C++20).

If that sentence meant nothing to you, good — this README assumes only that
you've finished an introductory C++ course. It explains the finance idea, the
C++ behind it, and how a single order actually moves through the code. Read it
top to bottom and you'll understand the whole project.

---

## 1. What problem does this solve?

An **exchange** (think a stock exchange) is a place where people who want to
buy meet people who want to sell. The piece of software that decides *who
trades with whom, and at what price* is called a **matching engine**, and the
data structure it keeps is called an **order book**. This project is a small
version of exactly that.

### The vocabulary you need (five words)

- **Order** — a request to trade. It has a **side** (buy or sell), a **price**
  (the most you'll pay, or the least you'll accept), and a **quantity** (how
  many units).
- **Bid** — a buy order sitting in the book, waiting.
- **Ask** (or *offer*) — a sell order sitting in the book, waiting.
- **Resting order** — an order that couldn't trade immediately, so it waits in
  the book until someone is willing to trade against it.
- **Trade** (or *fill*) — what happens when a buy and a sell agree. One order
  eats into the other's quantity.

### How matching works

When a new order arrives, the engine asks: *can this trade right now?*

- A new **buy** can trade if there's a **sell already resting at a price it's
  willing to pay** (i.e. a resting ask priced at or below the buy's price).
- A new **sell** can trade if there's a **buy already resting at a price it's
  willing to accept** (a resting bid priced at or above the sell's price).

If it can trade, it does — possibly against several resting orders at once. If
there's quantity left over that can't trade, that leftover **rests** in the
book and waits.

### "Price-time priority" — the matching rule

When several resting orders *could* fill an incoming order, which one goes
first? The rule almost every real exchange uses:

1. **Price first.** A buyer takes the cheapest available sell. A seller hits the
   highest available buy. Better price always wins.
2. **Time second.** Among orders at the *same* price, the one that arrived
   *first* trades first. Waiting in line matters.

That's the entire business logic. Everything in the code exists to enforce
those two rules quickly.

### A worked example

Say the book already holds two resting sell orders:

```
Sell 5 units @ price 100   (order A, arrived first)
Sell 5 units @ price 101   (order B, arrived second)
```

Now a **buy for 8 units @ price 101** arrives. It's willing to pay up to 101.

- **Price priority**: it takes the *cheapest* sell first → order A at 100.
  That fills 5 of its 8 units. (3 left.)
- Next cheapest is order B at 101. It fills the remaining 3 units there.
- Result: two trades (5 @ 100, then 3 @ 101), and order B has 2 units still
  resting.

The demo program (`apps/demo.cpp`) runs almost exactly this scenario so you can
see it happen.

---

## 2. The concepts this project demonstrates

Each of these is something an engineer at a trading firm is expected to reason
about. You meet them in order of difficulty.

- **Order book design** — choosing data structures so "what's the best price?",
  "add an order", and "cancel an order" are all cheap. This is the central
  design problem.
- **Price-time priority matching** — the algorithm above, implemented correctly.
- **Cache efficiency** — *how the data is laid out in memory* affects speed,
  sometimes a lot. Phase 3 rewrites the storage to be faster and **measures**
  the difference instead of guessing.
- **Low-latency thinking** — measuring honestly, and knowing where the time
  goes, rather than assuming.

Threads and lock-free structures are deliberately **not** here — they're listed
as future phases at the bottom, kept separate because they're much harder.

---

## 3. The C++ you'll see (and why it's used)

You don't need anything beyond a first course, but here's a map of the
less-obvious pieces so nothing is a surprise.

- **Integer prices.** Prices are stored as whole numbers (`std::int64_t`), not
  `double`. Decimals like 0.01 can't be represented *exactly* in floating
  point, and money bugs love that gap. So we count in whole "ticks" (think
  cents) instead. See `Types.hpp`.
- **`enum class Side { Buy, Sell }`.** A tiny type with exactly two values.
  Clearer and safer than passing a `bool` around.
- **`struct Order` / `struct Trade`.** Plain data bundles — just fields grouped
  together.
- **Standard containers.** The whole project is built out of the containers
  from your course. Which one is used where, and *why*, is the interesting part
  — see the next section.
- **`std::optional<Price>`.** `best_bid()` might have no answer (the book could
  be empty). Instead of returning a fake value like `-1`, it returns an
  `optional`: either a price, or "nothing" (`std::nullopt`). The caller checks
  with `.has_value()` or supplies a fallback with `.value_or(...)`.
- **References (`Order&`).** When the matching loop shrinks an order's
  quantity, it holds a **reference** to the real order in the book, so the
  change sticks. A copy would be thrown away.
- **A template + a lambda** in `OrderBook::match`. Don't worry about the
  machinery — it just lets one piece of code handle both "a buy crossing sells"
  and "a sell crossing buys" without duplicating it. You can read the two
  branches in `add()` and understand the whole flow without touching the
  template.

---

## 4. How the data is stored (the heart of the project)

Everything comes down to one question: **how do you store resting orders so the
best price is always easy to find?**

### Phase 1 version — `OrderBook` (simple and obviously correct)

Each side of the book (bids and asks) is a **`std::map`** keyed by price. A
`std::map` keeps its keys sorted automatically, so "the best price" is just the
first or last key.

At each price, orders are kept in a **`std::deque`** in arrival order — oldest
at the front. Since time priority means "oldest trades first," the engine
always takes from the front. 

There's also a **`std::unordered_map`** from order-id to its location, so
cancelling an order is a quick lookup instead of searching the whole book.

This version is easy to *read* and easy to *trust*. That's the point of Phase 1.

### Phase 3 version — `FlatOrderBook` (the same behaviour, faster)

Here's the subtle idea. A `std::map` is a tree, and its pieces are scattered all
over memory. Every time the CPU follows a link to another piece, it may have to
wait on a slow memory fetch (a "cache miss"). For a book where almost all the
action is at the few best prices, that scattering is wasted time.

So `FlatOrderBook` stores each side as a single **`std::vector`** of price
levels, kept sorted with the best price at the back. A vector is one
**contiguous** block of memory, which the CPU is very good at reading quickly.

**The honest tradeoff:** inserting a brand-new price in the *middle* of a vector
means shifting elements, which is slow. That's fine when new prices are rare —
but on a workload full of never-seen prices, this layout could actually *lose*.
Phase 3's real lesson is to **measure which effect wins**, not to assume the
"optimized" version is always better.

Both versions pass the **same test suite**, proving they behave identically —
the flat one is just (usually) faster. The benchmark runs them head-to-head.

---

## 5. Following one order through the code

To connect the idea to the files, here's what happens when you call
`book.add(Side::Buy, 101, 8, /*id=*/3)` on the Phase 1 book:

1. **`add()` in `src/OrderBook.cpp`** starts with `remaining = 8`.
2. It looks at the **asks** side (a buy trades against sells). It repeatedly
   takes the **cheapest** ask whose price is `<= 101`.
3. For each match it shrinks both the incoming order's `remaining` and the
   resting order's `quantity`, and records a `Trade`. A fully-eaten resting
   order is removed from the book.
4. It stops when either `remaining` hits 0 or no ask is cheap enough.
5. If `remaining` is still above 0, the leftover **rests**: it's added to the
   bids side at price 101, and registered in the id→location index.
6. The list of `Trade`s is returned to the caller.

Read `add()` with this list beside you and it should track line for line.

---

## 6. Prerequisites

- A C++20 compiler (GCC 11+, Clang 13+, or MSVC 2022).
- CMake 3.20 or newer.
- Git (the test framework, GoogleTest, is downloaded automatically — you don't
  install it yourself).

Nothing else. No libraries to install by hand.

---

## 7. Build and run

```bash
# 1. Configure: creates a build/ folder with the build setup inside.
cmake -S . -B build

# 2. Compile everything: the engine, tests, benchmark, and demo.
cmake --build build

# 3. Run the tests. Every one should pass.
ctest --test-dir build --output-on-failure

# 4. Run the demo: feeds a scripted set of orders and prints the trades.
./build/demo

# 5. Run the benchmark: times both book versions on the same workload.
./build/latency_bench
```

If step 3 shows all tests passing, the engine is behaving correctly. If the
benchmark's trade counts for the two books don't match, something is wrong —
that check is there on purpose (it once caught a real bug during development).

---

## 8. Where each file lives and what it does

```
include/orderbook/
  Types.hpp          Basic type names and the Side enum (Buy / Sell).
  Order.hpp          The Order struct: side, price, quantity, arrival number.
  Trade.hpp          The Trade struct: what a match produces.
  OrderBook.hpp      Phase 1 engine's public interface (std::map based).
  FlatOrderBook.hpp  Phase 3 engine's interface (std::vector based, faster).

src/
  OrderBook.cpp      Phase 1 matching logic. Start reading here.
  FlatOrderBook.cpp  Phase 3 matching logic (same behaviour, different storage).

tests/
  OrderBookTests.cpp      Correctness tests for the Phase 1 engine.
  FlatOrderBookTests.cpp  The SAME tests for Phase 3, proving they behave alike.
  CMakeLists.txt          Downloads GoogleTest and builds the test program.

bench/
  latency_bench.cpp  Times both engines on one shared workload and compares.

apps/
  demo.cpp           A tiny runnable example that prints orders and trades.

CMakeLists.txt       Top-level build configuration.
```

**Suggested reading order:** `Types.hpp` → `Order.hpp` → `Trade.hpp` →
`OrderBook.hpp` → `src/OrderBook.cpp` (with section 5 above beside you) →
`tests/OrderBookTests.cpp` → then the Phase 3 pair.

---

## 9. Roadmap (what's done, what's future)

### Phase 1 — Single-threaded core ✅ (done)
A correct matching engine with price-time priority, partial fills, and cancels,
backed by a full test suite. This alone is a complete, presentable project.

### Phase 2 — Measure ✅ (done)
A benchmark that times the engine honestly. The skill here is measuring only the
thing you care about and reading the number without fooling yourself.

### Phase 3 — Cache-friendly rework ✅ (done)
`FlatOrderBook`: same behaviour, memory laid out for speed, with a head-to-head
benchmark and shared tests proving equivalence. This is the most interesting
part to discuss, because it's about *tradeoffs*, not just "make it faster."

### Phase 4 — Concurrency (future, harder)
Split the work across two threads (one feeding orders, one matching them),
connected by a thread-safe queue. Requires solid threading fundamentals.

### Phase 5 — Lock-free handoff (future, hardest)
Replace that queue with a lock-free one using atomics and memory ordering. This
is genuinely advanced — only worth attempting once Phases 1–4 are rock solid,
and only if you can explain every line of it.