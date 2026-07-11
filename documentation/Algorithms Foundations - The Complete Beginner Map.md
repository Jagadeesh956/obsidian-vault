---
title: "Algorithms Foundations - The Complete Beginner Map"
date: 2026-07-10
tags: [algorithms, patterns, complexity, big-o, data-structures, beginner, index]
summary: "One map connecting numbers/logarithms -> asymptotic notation -> data structures -> patterns -> real-world systems, with plain-English terminology and worked examples throughout."
---

# Algorithms Foundations — The Complete Beginner Map

> **Purpose of this note:** everything else in this vault ([[How Patterns Fit Problems and Algorithm Complexity]], [[Deciding Coding Patterns by Input Data Type]]) assumes you already know what "O(n log n)" or "heap" *means*. This note is the missing layer underneath — the terminology, the math, and the real-world mapping — so those notes become easy to read.

---

## 0) Index

1. [[#1) How to Use This Document]]
2. [[#2) The Big Picture — One Sentence Per Concept]]
3. [[#3) Terminology Glossary — Plain English First]]
4. [[#4) Numbers and Logarithms — The Math You Actually Need]]
5. [[#5) Asymptotic Notation — Big O, Big Omega, Big Theta]]
6. [[#6) Growth Rates With Real Numbers]]
7. [[#7) Data Structures Mapped to Real Life]]
8. [[#8) Sorted vs Unsorted — Why It Keeps Coming Up]]
9. [[#8.5) The Space-Time Tradeoff — Why Any of This Actually Works]]
10. [[#9) Pattern Catalog — Real-World Use Case for Each]]
11. [[#10) The Full Decision Flow — Problem to Production System]]
12. [[#11) Worked End-to-End Example]]
13. [[#12) Study Roadmap]]
14. [[#13) Resources]]
15. [[#14) Alphabetical Glossary]]

---

## 1) How to Use This Document

Read top to bottom the first time. After that, use it as a lookup — jump straight to the section you're fuzzy on via the index links above (Obsidian: click the `[[#Heading]]` link, or `Ctrl/Cmd+Click`).

The learning order that actually works, in order:

```text
Numbers & logs (section 4)
   -> Asymptotic notation (section 5)
      -> Data structures (section 7)
         -> Space-time tradeoff (section 8.5)
            -> Patterns (section 9)
               -> Real systems (section 10-11)
```

Skipping straight to "patterns" without sections 4-5 is why terms like `O(n log n)` feel like magic. Do it in order once.

---

## 2) The Big Picture — One Sentence Per Concept

| Concept | One-sentence definition |
|---|---|
| **Algorithm** | A finite, step-by-step procedure that turns an input into an output. |
| **Data structure** | A specific way of organizing data in memory so certain operations are fast. |
| **Pattern** | A reusable strategy (e.g. two pointers, sliding window) that solves a *family* of problems, not just one. |
| **Complexity** | A measurement of how an algorithm's resource usage (time or memory) grows as input size grows. |
| **Asymptotic notation** | The mathematical language (Big O, etc.) used to describe that growth, ignoring constants and small inputs. |
| **Logarithm** | The answer to "how many times do I halve (or divide) this number before I reach 1?" |

Everything below expands each of these one at a time, then connects them.

---

## 3) Terminology Glossary — Plain English First

Read this before section 4-5; it removes the jargon wall.

- **Input size (`n`)** — how many items you're processing (array length, string length, number of nodes, etc.)
- **Operation** — one basic unit of work (a comparison, an addition, a memory access). Complexity counts these, not seconds.
- **Time complexity** — how the *number of operations* grows as `n` grows.
- **Space complexity** — how much *extra memory* (beyond the input itself) grows as `n` grows.
- **Worst case** — the input that makes the algorithm do the most work (this is what Big O usually describes).
- **Average case** — expected work over "typical" random inputs.
- **Best case** — the friendliest possible input (rarely the number people quote).
- **Amortized** — "expensive sometimes, cheap on average over many operations" (e.g. a dynamic array occasionally resizing).
- **In-place** — an algorithm that doesn't need extra memory proportional to `n` (space complexity `O(1)`).
- **Stable sort** — a sort that keeps equal elements in their original relative order.
- **Recursion** — a function that calls itself on a smaller version of the same problem.
- **Base case** — the smallest version of a recursive problem, solved directly without further recursion (stops infinite recursion).
- **Divide and conquer** — split a problem into smaller independent pieces, solve each, combine results. (See merge sort trace in [[How Patterns Fit Problems and Algorithm Complexity]].)
- **Greedy** — make the locally-best choice at each step and never look back, hoping it leads to the global best.
- **Dynamic Programming (DP)** — break a problem into overlapping subproblems and cache (memoize) results so you never recompute the same thing twice.
- **Heuristic** — a rule of thumb that gives a good-enough answer fast, without guaranteeing the optimal one.

---

## 4) Numbers and Logarithms — The Math You Actually Need

You do **not** need a math degree. You need one idea, deeply understood: **logarithm = "how many times can I halve this before I reach 1?"**

### 4.1 Build it from scratch

```text
Start with n = 16
16 -> 8 -> 4 -> 2 -> 1      (4 halvings)
```

We just computed `log2(16) = 4`. That's it — that's the whole definition for our purposes.

### 4.2 A lookup table to build intuition

| n | halvings to reach 1 (`log2 n`) |
|---|---|
| 2 | 1 |
| 4 | 2 |
| 8 | 3 |
| 16 | 4 |
| 1,024 | 10 |
| 1,000,000 | ~20 |
| 1,000,000,000 | ~30 |

**The pattern to internalize:** `n` grows by 1000x (1,024 → 1,000,000 → 1,000,000,000) but `log2 n` only grows by about 10 each time. Logarithms grow *unbelievably* slowly compared to `n`. This single fact is why every "log n" algorithm (binary search, balanced trees, heaps) is so powerful at scale.

### 4.3 Where `log n` shows up

Any process that **repeatedly cuts the problem in half** produces a `log n` term:

- Binary search: halves the search range each step
- Merge sort: halves the array each split
- Balanced binary trees (e.g. lookups in a sorted map): halves the remaining subtree each step
- Heaps: height of the tree is `log n`, so insert/remove is `O(log n)`

### 4.4 Why base doesn't matter in Big-O

```text
log2(n) = log10(n) / log10(2) = log10(n) x 3.32...
```

Changing the base just multiplies by a constant. Big-O ignores constant multipliers (see section 5), so we always just write `log n` without specifying a base.

### 4.5 Where logarithms came from

Logarithms weren't invented from a single formula out of nowhere — they came from a specific problem: 17th-century astronomers needed to multiply huge numbers by hand, and multiplication was slow. **John Napier** (1614) found that logarithms turn multiplication into addition:

```text
log(a x b) = log(a) + log(b)
```

Formal definition (the one used throughout this doc):

```text
if  b^y = x   then   log_b(x) = y
```

A logarithm is the **inverse of exponentiation** — "what power do I raise `b` to, to get `x`?" Applied to repeated halving (`2^k = n`), this is exactly where `O(log n)` comes from in binary search, heaps, and balanced trees.

---

## 5) Asymptotic Notation — Big O, Big Omega, Big Theta

### 5.1 What "asymptotic" means

*Asymptotic* = "what happens as `n` approaches infinity/gets very large" — i.e. we care about **growth trend**, not exact numbers, and definitely not behavior on tiny inputs.

### 5.2 The three notations (people mostly only say "Big O", but here's the full family)

| Notation | Meaning | Plain English |
|---|---|---|
| **Big O** `O(f(n))` | Upper bound | "This algorithm takes *at most* roughly `f(n)` operations" (worst case ceiling) |
| **Big Omega** `Ω(f(n))` | Lower bound | "This algorithm takes *at least* roughly `f(n)` operations" (best case floor) |
| **Big Theta** `Θ(f(n))` | Tight bound | "This algorithm takes *exactly* roughly `f(n)` operations" (upper and lower bound match) |

In everyday interviews/engineering, people say "Big O" loosely to mean "roughly how this scales," even when they technically mean Theta. That's fine for practical use — just know the distinction exists.

### 5.3 Why constants get dropped

**Sample:** An algorithm that does exactly `5n + 3` operations.

```text
n = 10:         5(10) + 3 = 53 operations
n = 1,000:      5(1000) + 3 = 5,003 operations
n = 1,000,000:  5(1,000,000) + 3 = 5,000,003 operations
```

As `n` grows, the `+3` becomes irrelevant, and the `5x` multiplier doesn't change the *shape* of growth (still a straight line). So we call this `O(n)` — we drop the `5` and the `3` because they don't change how it scales, only a constant "speed" that varies by hardware/language anyway.

### 5.4 Full derivation of a real one (linked from your earlier question)

We already derived merge sort's exact cost:

```text
T(n) = cn·log(n) + c0·n     (exact formula, with constants)
```

Simplified to `O(n log n)` by dropping the constant `c`, the lower-order term `c0·n`, and the log base. Full step-by-step expansion of the recurrence lives in [[How Patterns Fit Problems and Algorithm Complexity]] (section 7.4) — this note gives you the *why we're allowed to drop things* reasoning; that note gives the *mechanical derivation*.

---

## 6) Growth Rates With Real Numbers

This table is the single most useful reference in this whole document. Memorize its **shape**, not the exact numbers.

| n | O(1) | O(log n) | O(n) | O(n log n) | O(n²) | O(2ⁿ) |
|---|---|---|---|---|---|---|
| 10 | 1 | ~3 | 10 | ~33 | 100 | 1,024 |
| 100 | 1 | ~7 | 100 | ~664 | 10,000 | ~1.3 x 10³⁰ |
| 1,000 | 1 | ~10 | 1,000 | ~9,966 | 1,000,000 | astronomically large |
| 1,000,000 | 1 | ~20 | 1,000,000 | ~20,000,000 | 10¹² | impossible to compute |

**Takeaways:**
- `O(1)` and `O(log n)` barely move even at huge `n` — this is why hash maps and binary search feel "instant."
- `O(n²)` becomes unusable past a few thousand items (nested loops on large data = the classic beginner performance bug).
- `O(2ⁿ)` (naive recursion without memoization, e.g. unoptimized Fibonacci or brute-force subsets) becomes impossible past `n ≈ 40`, regardless of hardware.

---

## 7) Data Structures Mapped to Real Life

This is the section that makes it click — each structure exists because some real system needed a specific operation to be fast.

| Data structure | Real-world analogy | Why it's shaped that way |
|---|---|---|
| **Array** | A row of numbered mailboxes / a bookshelf with fixed slots | Instant access by position (`O(1)`) because you can calculate exactly where slot #47 is |
| **Linked List** | A **music playlist** — each song only knows "what's next," you can't jump to song #50 without walking through 1-49 | Cheap to insert/remove a song in the middle (just rewire two pointers), but no random access (`O(n)` to reach position k) |
| **Hash Map / Hash Set** | A **phonebook indexed by name** — but a *smart* one that computes exactly which page to open, instead of flipping pages | `O(1)` average lookup/insert because a hash function converts the key directly into a memory location |
| **Sorted array + Binary Search** | An **old-school paper phonebook sorted alphabetically**, or a **physical dictionary** — you open to the middle, see you overshot, flip back half, repeat | `O(log n)` lookup because sorted order lets you eliminate half the remaining pages each guess |
| **Stack** | A **stack of plates** — you can only add/remove from the top | Used for undo/redo, function call tracking (call stack), matching brackets `()` |
| **Queue** | A **checkout line at a store** — first person in line is served first | Used for task scheduling, printer job queues, BFS traversal order |
| **Binary Search Tree / Balanced Tree** | A **company org chart**, or an **auto-organizing filing cabinet** that keeps itself sorted as you add files | `O(log n)` search/insert/delete while staying sorted at all times (databases' B-trees are a cousin of this) |
| **Heap (priority queue)** | A **hospital ER** — patients aren't served first-come-first-served, the most critical case always goes next | `O(log n)` insert, `O(1)` peek-at-most-urgent — used for task schedulers, Dijkstra's shortest path, "top-k" problems |
| **Graph** | A **road map** or a **social network** (people = nodes, friendships = edges) | Models relationships/connections that arrays and trees can't represent (cycles, many-to-many links) |
| **Trie (prefix tree)** | The **autocomplete dropdown in your phone's keyboard** | `O(k)` lookup where `k` = word length, by sharing common prefixes across many words |

---

## 8) Sorted vs Unsorted — Why It Keeps Coming Up

Full deep-dive already exists here: [[Deciding Coding Patterns by Input Data Type]] — this section is just the real-world anchor for it.

- **Sorted → binary search analogy:** a **paper phonebook** sorted by last name. You never scan linearly; you jump to the middle and eliminate half each time. `O(log n)`.
- **Unsorted → hash map analogy:** your **phone's contacts app**. Contacts aren't stored alphabetically in memory — the app builds an index (hash map) so tapping a name feels instant regardless of order. `O(1)` average.
- **Unsorted, need top-k → heap analogy:** a **"Trending Now" feed** — you don't sort *all* posts by popularity, you maintain a small heap of the current top 10, updated as new posts arrive.
- **Streaming, can't store all → running-stats analogy:** a **live stock ticker** computing a running average/median without keeping every historical tick in memory.

---

## 8.5) The Space-Time Tradeoff — Why Any of This Actually Works

This is the single idea that unifies section 7 (data structures) and section 9 (patterns) into one economic principle instead of a list of separate tricks.

### 8.5.1 The core idea

> **Do extra work once, at insert/build time, so that every future query pays less.**

Every "smart" data structure is a bet: pay more now, pay less later, and win overall if you query enough times.

### 8.5.2 Proof across every structure in section 7

| Structure | Work done on insert/build | What that buys you on query |
|---|---|---|
| Hash map | Compute a hash, place in the right bucket — `O(1)` per insert | `O(1)` average lookup — jump straight to the bucket, no scanning |
| Sorted array | Sort the whole array once — `O(n log n)` | `O(log n)` binary search instead of `O(n)` linear scan |
| Balanced BST | Rebalance on every insert — `O(log n)` per insert | `O(log n)` search, guaranteed, regardless of insert order |
| Heap | "Bubble up" new item — `O(log n)` per insert | `O(1)` peek at min/max instantly |
| Trie | Walk/create nodes per character — `O(k)` per word | `O(k)` prefix lookup, independent of how many other words exist |
| Database index (B-tree) | Build the index — expensive one-time cost | Queries drop from scanning millions of rows to `O(log n)` |

In every row: insert is strictly more expensive than "just append it somewhere" — and that extra expense is exactly what makes reads cheap later.

### 8.5.3 Two different levers — don't conflate them

Not all speed comes from paying at insert time. There are two distinct levers:

```text
Lever A: exploit a property the input ALREADY has (sorted order, contiguity)
         -> no preprocessing needed
         -> e.g. two pointers on a sorted array, sliding window over a contiguous range

Lever B: pay upfront to CREATE a property the input doesn't have
         -> preprocessing cost paid once
         -> e.g. sorting before binary search, building a hash map, building a trie
```

Both levers reduce query-time cost. The difference is only *where* the cheap property came from — already present (Lever A) vs. manufactured by you (Lever B).

### 8.5.4 The formula that decides if preprocessing is worth it

```text
total_cost = preprocessing_cost + (num_queries x cost_per_query)
```

**Worked example — is sorting worth it before repeated searches?**

```text
n = 1,000,000 items

Option 1 (no sort, linear search each time):
   each search = O(n) = 1,000,000 ops

Option 2 (sort once, then binary search):
   sort = O(n log n) ≈ 20,000,000 ops        (one-time)
   each search after = O(log n) ≈ 20 ops

Search once:    Option 1 wins  -> 1,000,000  <  20,000,020
Search 1,000x:  Option 2 wins  -> 1,000,000,000  vs  ~20,020,000
```

This exact tradeoff — "is it worth building an index/cache/hash map for how often this will be queried?" — is the real engineering decision behind database indexes, application caches, and every hash map used in production systems.

### 8.5.5 The one question that resolves almost everything in sections 7 and 9

> **When you see a data structure or pattern that's "smarter" than a plain array scan, ask: what did it pay at insert time, and what does that buy at query time? If the answer is "nothing was paid," ask instead: what property does the input already have that's being exploited?**

Keep this question in mind while reading section 9 below — for each pattern, try to answer it yourself before reading the table's "real-world system" column.

---

## 9) Pattern Catalog — Real-World Use Case for Each

Cross-reference: full code examples for most of these already exist in [[How Patterns Fit Problems and Algorithm Complexity]] and [[Deciding Coding Patterns by Input Data Type]]. This table adds the **"where is this actually used"** layer.

| Pattern | Sample input | Real-world system that uses it |
|---|---|---|
| **Binary Search** | Sorted array `[2,5,8,12,16,23]` | Dictionary/phonebook lookup, `git bisect` (finding which commit broke something), searching sorted database indexes |
| **Two Pointers** | Sorted array, find pair summing to target | Detecting cycles in a linked list (fast/slow pointer), palindrome checking, merging sorted result sets |
| **Sliding Window** | `[4,2,1,7,8,1,2,8,1,0]`, window size 3 | Netflix/YouTube "trending in last 24h" calculations, network rate limiting (requests per minute window), stock max-profit-in-window |
| **Hash Map / Hash Set** | `["apple","banana","apple"]` frequency count | Browser cache/deduplication, spell-checkers, database indexing, contacts app lookup |
| **Heap / Priority Queue** | `[7,10,4,3,20,15]`, get top 3 | Hospital ER triage, OS task scheduler (highest priority process runs next), Uber/Lyft driver-matching by proximity, Dijkstra's shortest path |
| **BFS (Breadth-First Search)** | Graph of city connections | GPS "shortest number of hops" routing, social network "people you may know" (friends of friends), web crawlers |
| **DFS (Depth-First Search)** | Same graph, explore-deep-first | File system search (walking directory trees), maze solving, detecting cycles/dependencies (e.g. circular imports) |
| **Dynamic Programming** | Fibonacci, edit distance | Autocorrect/spellcheck (Levenshtein edit distance), GPS route cost optimization, DNA sequence alignment in bioinformatics |
| **Greedy** | Interval scheduling sorted by end time | Meeting room schedulers, Huffman compression (used in zip/gzip/JPEG), coin change with "nice" denominations |
| **Divide and Conquer** | Merge sort on `[8,3,7,1,9,2,6,4]` | Sorting large datasets, `numpy`/database sort implementations, closest-pair-of-points geometry problems |
| **Union-Find (Disjoint Set)** | Group of connected accounts/IDs | Social network "are these two people in the same friend group," Kruskal's minimum spanning tree, detecting duplicate accounts |
| **Trie** | Words: `cat, car, cart, dog` | Autocomplete suggestions, IDE code-completion, spell-checkers |

---

## 10) The Full Decision Flow — Problem to Production System

This ties every section above into one flow you can literally follow when facing a new problem:

```text
STEP 1: What does the input look like?
   -> sorted? unsorted? streaming? relational (graph)? See section 7-8.

STEP 2: What's the shape of the question?
   -> existence/lookup? top-k? contiguous range? shortest path? optimal combination?
      See section 9 (pattern catalog).

STEP 3: Pick the pattern that exploits the input property from Step 1
   for the question shape in Step 2 — and check section 8.5:
   does this pattern need preprocessing (Lever B) or exploit existing structure (Lever A)?

STEP 4: Express its cost using asymptotic notation (section 5-6),
   sanity-check against the growth table (section 6) for your actual n.

STEP 5: Map it to a real system you already know (section 7, 9)
   to confirm your intuition — "oh, this is basically what a phonebook/
   contacts app/GPS app does."
```

---

## 11) Worked End-to-End Example

**Problem:** "Given a list of usernames typed into a search box as the user types, suggest matching usernames instantly."

Walking the flow from section 10:

```text
Step 1 (input):     A fixed, large list of usernames. Not sorted by prefix usefully.
Step 2 (question):  "starts with X" lookups, repeated on every keystroke.
Step 3 (pattern):   Trie (prefix tree) — built once (Lever B, section 8.5), then O(k) lookup per keystroke,
                     k = length of typed prefix.
Step 4 (cost):      O(k) per lookup instead of O(n) scanning the whole list every keystroke.
                     At n = 1,000,000 usernames, that's the difference between
                     ~20 character comparisons vs. a million string comparisons, per keystroke.
Step 5 (real system): This is exactly how phone keyboard autocomplete and
                     IDE code-completion work (section 9, Trie row).
```

---

## 12) Study Roadmap

A practical order to actually get fluent, week by week:

```text
Week 1: Sections 4-6 of this note. Get log n and Big-O until it's boring/obvious.
Week 2: Section 7 — implement array, linked list, hash map, stack, queue by hand once each.
         Then read section 8.5 and identify each one's insert-cost vs query-cost tradeoff yourself.
Week 3: Binary search, two pointers, sliding window (all rely on section 4's log/linear intuition).
Week 4: Hash map pattern + heap pattern (top-k, frequency problems).
Week 5: BFS/DFS on graphs (use section 7's road-map/social-network analogy while coding).
Week 6: Recursion -> Divide and Conquer -> Dynamic Programming, in that order (each builds on the last).
Ongoing: For every problem you solve, run it through section 10's 5-step flow explicitly,
         in writing, until it becomes automatic.
```

---

## 13) Resources

Tag: `#resources`

- [Big-O Cheat Sheet](https://www.bigocheatsheet.com/) — quick lookup of complexity for common data structures/algorithms
- [VisuAlgo](https://visualgo.net/) — animated visualizations of sorting, graph traversal, trees, heaps
- [CS50's Introduction to Computer Science (Harvard, free)](https://cs50.harvard.edu/x/) — strong beginner-friendly coverage of these exact fundamentals
- [Python `time` complexity reference (official wiki)](https://wiki.python.org/moin/TimeComplexity) — real complexity of Python's built-in structures (list, dict, set)

Internal vault links:
- [[How Patterns Fit Problems and Algorithm Complexity]] — pattern decision flow + full merge sort/recurrence derivation
- [[Deciding Coding Patterns by Input Data Type]] — deep dive on sorted vs unsorted input driving pattern choice
- [[How to Approach a Programming Problem]] — general problem-solving approach

---

## 14) Alphabetical Glossary

- **Amortized complexity** — average cost per operation across a sequence, when individual operations vary in cost (see section 3)
- **Asymptotic notation** — math language describing growth trends (section 5)
- **Base case** — the stopping condition of a recursive function (section 3)
- **Big O / Big Omega / Big Theta** — upper bound / lower bound / tight bound notations (section 5.2)
- **Binary search** — halving search pattern for sorted data (sections 4.3, 9)
- **Divide and conquer** — split, solve, combine strategy (sections 3, 9)
- **Dynamic programming** — caching overlapping subproblem results (sections 3, 9)
- **Greedy algorithm** — locally-optimal-choice strategy (sections 3, 9)
- **Hash map** — key-based O(1) average lookup structure (sections 7, 9)
- **Heap** — priority-ordered tree structure (sections 7, 9)
- **In-place** — algorithm using O(1) extra space (section 3)
- **Logarithm** — "how many halvings to reach 1" (section 4)
- **Recursion** — function calling itself on smaller input (section 3)
- **Space complexity** — extra memory growth relative to input size (section 3)
- **Space-time tradeoff** — paying more work at insert/build time to reduce work at query time (section 8.5)
- **Stable sort** — preserves relative order of equal elements (section 3)
- **Time complexity** — operation count growth relative to input size (section 3)
- **Trie** — prefix-sharing tree for string lookups (sections 7, 9)
- **Two pointers** — pattern using two indices moving through sorted/linear data (section 9)

---

## Final Note

The terminology only feels overwhelming because each term is usually taught in isolation. The moment you can trace **numbers → logs → Big-O → data structure → space-time tradeoff → pattern → real system** as one continuous chain (section 10's flow), the vocabulary stops being scary — it becomes labels for things you already understand intuitively from apps you use every day.
