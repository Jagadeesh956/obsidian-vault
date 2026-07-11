---
title: "How Patterns Fit Problems and Algorithm Complexity"
date: 2026-07-03
tags: [algorithms, recursion, patterns, complexity, problem-solving]
summary: "How to choose the right solving pattern, with examples, and how to reason about time and space complexity in practical terms."
---

# How Patterns Fit Problems and Algorithm Complexity

This note answers three practical questions:

1. **How do I know which pattern fits a problem?**
2. **How do common patterns work with examples?**
3. **What is algorithm complexity and how do I calculate it?**

---

## 1) Why Patterns Matter

Most coding problems are not fully new.
They usually belong to a known *pattern family*.

If you can detect the pattern early, you save time and avoid random trial-and-error.

Think of patterns like tools:
- recursion is one tool
- two pointers is another
- dynamic programming is another

You pick the tool based on the *shape* of the problem.

---

## 2) How to Decide Which Pattern Fits

Use this quick decision flow:

```text
A) Is the problem naturally split into smaller same-type subproblems?
   -> Try recursion / divide-and-conquer / DP

B) Is input sequential (array/string) and you need ranges/windows?
   -> Try two pointers / sliding window / prefix sums

C) Need fast lookups or duplicate checks?
   -> Try hash map / hash set

D) Need best/min/max repeatedly while processing stream?
   -> Try heap / priority queue

E) Problem is about connections (nodes/edges/dependencies)?
   -> Try graph (BFS/DFS/topological sort)

F) Need "best" answer from many choices with overlap?
   -> Try dynamic programming
```

---

## 3) Pattern-by-Pattern: When and Example

## 3.1 Recursion

### When recursion fits

Use recursion when:
- problem definition is self-referential
- large problem can be solved by solving smaller same problem
- tree/graph traversal feels natural top-down

### Mental model

Recursion = function calls itself with smaller input until base case.

### Example: Factorial

```python
def factorial(n):
    if n <= 1:
        return 1
    return n * factorial(n - 1)
```

- Base case: `n <= 1`
- Recursive case: `n * factorial(n-1)`

### Common recursion mistakes

- missing base case -> infinite recursion
- base case not reachable
- repeated work (can be optimized with memoization)

---

## 3.2 Two Pointers

### When it fits

Use when:
- array/string can be scanned from left/right
- target depends on pair/segment
- sorted input gives leverage

### Example: Pair sum in sorted array

```python
def has_pair_sum(arr, target):
    i, j = 0, len(arr) - 1
    while i < j:
        s = arr[i] + arr[j]
        if s == target:
            return True
        if s < target:
            i += 1
        else:
            j -= 1
    return False
```

---

## 3.3 Sliding Window

### When it fits

Use when:
- problem asks for best/longest/shortest subarray or substring
- constraints are local to a contiguous range

### Example: Longest substring without repeating chars

Core idea:
- expand right pointer
- shrink left pointer until valid
- track best length

---

## 3.4 Hash Map / Hash Set

### When it fits

Use when:
- frequent existence checks needed
- counting frequencies
- mapping values to positions

### Example: Two Sum

```python
def two_sum(nums, target):
    seen = {}
    for i, x in enumerate(nums):
        need = target - x
        if need in seen:
            return [seen[need], i]
        seen[x] = i
```

---

## 3.5 Divide and Conquer

### When it fits

Use when:
- can split problem into independent parts
- combine results later

### Example: Merge Sort

- split array in half recursively
- sort both halves
- merge sorted halves

Time is `O(n log n)` because each level processes `n`, and levels are about `log n`.

---

## 3.6 Dynamic Programming (DP)

### When it fits

Use when problem has:
1. overlapping subproblems
2. optimal substructure

### Example: Fibonacci with memoization

```python
def fib(n, memo={}):
    if n in memo:
        return memo[n]
    if n <= 1:
        return n
    memo[n] = fib(n-1, memo) + fib(n-2, memo)
    return memo[n]
```

Without memoization: exponential time
With memoization: linear time

---

## 3.7 BFS / DFS (Graphs)

### When it fits

Use for:
- connectivity
- shortest path in unweighted graph (BFS)
- cycle detection / traversal (DFS)

### Example: BFS shortest steps in grid

- push start cell in queue
- pop, explore neighbors
- first time reaching target = shortest steps

---

## 3.8 Greedy

### When it fits

Use when local best choices guarantee global best.

### Example: Activity selection / interval scheduling

- sort by earliest end time
- always pick next compatible interval

Greedy is powerful but only when correctness can be justified.

---

## 4) How Algorithms Work (Simple View)

Any algorithm is:

1. **State** (data currently known)
2. **Transitions** (how state changes each step)
3. **Stopping condition**

If these three are clear, algorithm becomes easy to reason about.

Example (BFS):
- State: queue + visited set
- Transition: pop node, push unvisited neighbors
- Stop: queue empty or target found

---

## 5) What Complexity Means

Complexity tells how resource usage grows with input size `n`.

Two main dimensions:

- **Time complexity** -> number of operations as `n` grows
- **Space complexity** -> extra memory used as `n` grows

This is about **growth trend**, not exact microseconds.

---

## 6) Big-O Quick Rules

Common growth orders (best to worst):

- `O(1)` constant
- `O(log n)` binary search style
- `O(n)` single pass
- `O(n log n)` efficient sorting
- `O(n^2)` nested loops over same `n`
- `O(2^n)` exponential (subsets/naive recursion)
- `O(n!)` permutations brute force

---

## 7) How to Calculate Time Complexity

## 7.1 Count loops

- one loop over `n` -> `O(n)`
- two nested loops over `n` -> `O(n^2)`

```python
for i in range(n):
    for j in range(n):
        work()
```

## 7.2 Sequential blocks: add, then keep dominant term

`O(n) + O(n^2)` -> `O(n^2)`

## 7.3 Branches: keep worst branch

```python
if condition:
    O(n)
else:
    O(log n)
```
Worst case -> `O(n)`

## 7.4 Recursion: recurrence relation

Example merge sort:

`T(n) = 2T(n/2) + O(n)` -> `O(n log n)`

## 7.5 Amortized complexity

Some operations are occasionally expensive but average cheap.
Example: dynamic array append average `O(1)`.

---

## 8) How to Calculate Space Complexity

Count extra memory (not input itself unless required):

- few variables -> `O(1)`
- extra array size `n` -> `O(n)`
- recursion stack depth `n` -> `O(n)`
- recursion depth `log n` -> `O(log n)`

Example:

```python
def sum_array(arr):
    s = 0
    for x in arr:
        s += x
    return s
```

Extra space is `O(1)`.

---

## 9) Small Worked Examples (Time + Space)

## Example A: Linear search

```python
def linear_search(arr, target):
    for x in arr:
        if x == target:
            return True
    return False
```

- Time: `O(n)` worst case
- Space: `O(1)`

## Example B: Binary search (sorted input)

- Time: `O(log n)`
- Space: `O(1)` iterative, `O(log n)` recursive (call stack)

## Example C: Frequency count

```python
def freq(arr):
    m = {}
    for x in arr:
        m[x] = m.get(x, 0) + 1
    return m
```

- Time: `O(n)` average
- Space: `O(k)` where `k` = distinct elements (worst `O(n)`)

---

## 10) Pattern + Complexity Cheat Sheet

| Pattern | Typical Time | Typical Space | Use When |
|---|---|---|---|
| Recursion (naive) | varies, can be high | stack depth | natural subproblems |
| Recursion + memo (DP) | often `O(n)` | `O(n)` | repeated subproblems |
| Two pointers | `O(n)` | `O(1)` | sorted/contiguous scans |
| Sliding window | `O(n)` | `O(1)` or map | contiguous constraints |
| Hashing | `O(n)` avg | `O(n)` | fast lookup/count |
| Merge sort | `O(n log n)` | `O(n)` | stable sort |
| Heap top-k | `O(n log k)` | `O(k)` | top-k/priority |
| BFS/DFS | `O(V+E)` | `O(V)` | graph traversal |

---

## 11) Practical Study Habit

For each new problem, ask and write down:

1. Which pattern did I choose?
2. Why that pattern?
3. Time complexity and why?
4. Space complexity and why?
5. What edge case almost broke my solution?

Doing this repeatedly builds strong intuition fast.

---

## Final Note

Do not memorize 500 solutions.
Memorize:
- how to detect patterns
- how to model state transitions
- how to compute time/space cost

That is what makes you good in interviews **and** real-world engineering.
