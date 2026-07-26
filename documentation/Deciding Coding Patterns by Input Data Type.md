---
title: Deciding Coding Patterns by Input Data Type
date: 2026-07-10
tags:
  - algorithms
  - patterns
  - problem-solving
  - data-structures
summary: How the shape and properties of your input data (sorted, unsorted, linked, graph, stream, etc.) point you to the right coding pattern, with worked examples and sample inputs for each.
---

# Deciding Coding Patterns by Input Data Type

Related note: [[How Patterns Fit Problems and Algorithm Complexity]]

That note covers pattern detection from the *problem's* shape. This note comes at it from the other direction: starting from the **input data's properties**, what pattern does that property unlock or rule out?

---

## 1) The Core Idea

Every input has properties beyond just "it's an array" or "it's a string":

- Is it **sorted** or **unsorted**?
- Is it **static** (given once) or a **stream** (arrives over time)?
- Is it **linear** (array/list) or **relational** (graph/tree)?
- Do you need **one pass** or **repeated queries**?
- Is duplication / frequency information important?

The property you can exploit determines which pattern gives you better-than-brute-force performance. If you ignore the property, you default to `O(n)` or `O(n^2)` scans even when a smarter tool was available.

---

## 2) Sorted Data → Binary Search Family

### Why sorted data matters

Sorting encodes an **ordering guarantee**: `arr[i] <= arr[i+1]`. That guarantee lets you eliminate half the search space on every comparison, instead of checking every element.

### Sample input

```text
arr = [2, 5, 8, 12, 16, 23, 38, 45, 56, 72]
target = 23
```

### Pattern: Binary Search

```python
def binary_search(arr, target):
    lo, hi = 0, len(arr) - 1
    while lo <= hi:
        mid = (lo + hi) // 2
        if arr[mid] == target:
            return mid
        elif arr[mid] < target:
            lo = mid + 1
        else:
            hi = mid - 1
    return -1
```

- Time: `O(log n)` vs `O(n)` for linear scan
- Space: `O(1)`
- Works because each comparison discards half the remaining range

### Variants that also rely on "sorted"

| Variant | Sample input | What it solves |
|---|---|---|
| Two pointers (sorted) | `arr = [1,3,4,6,9]`, target sum `10` | Pair with target sum, in `O(n)` |
| Binary search on answer | `capacity range [1..1000]` | Minimize/maximize a feasible value |
| Sorted merge | `a=[1,4,7]`, `b=[2,3,9]` | Merge two sorted lists in `O(n+m)` |

**Rule of thumb:** if the input is sorted (or can be cheaply sorted and sorting doesn't destroy needed info like original index), always ask "can I binary search or two-pointer this?" before writing a linear/brute-force scan.

---

## 3) Unsorted Data — the Real Question

Unsorted data removes the ordering guarantee, so binary search's "eliminate half" trick no longer applies (you can't know which half the target is in). The decision now shifts to: **what other property can I exploit instead?**

Ask these in order:

```text
1. Can I sort it first? (if I don't need original order and n is small/medium)
   -> sort, then use binary search / two pointers -> O(n log n)

2. Do I need fast existence/frequency checks?
   -> hash map / hash set -> O(n) time, O(n) space

3. Do I only need the top-k / min / max repeatedly?
   -> heap (priority queue) -> O(n log k)

4. Do I need a contiguous range/subarray answer?
   -> sliding window / prefix sums -> O(n)

5. Is the data arriving as a stream (can't store it all)?
   -> running stats, reservoir sampling, streaming heap -> O(1) or O(log k) per item

6. Are elements related to each other (graph/dependency)?
   -> BFS/DFS/Union-Find -> O(V+E)
```

### 3.1 Option A — Sort first, then reuse sorted-data patterns

**Sample input**

```text
arr = [23, 5, 45, 2, 16, 12, 8]
target_pair_sum = 28
```

```python
def has_pair_sum(arr, target):
    arr.sort()                    # O(n log n)
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

Use this when: you don't care about original ordering/index, and `O(n log n)` is acceptable.

### 3.2 Option B — Hash Map / Hash Set (no sort needed)

Best when you need **O(n)** total time and order doesn't help you anyway.

**Sample input**

```text
nums = [2, 7, 11, 15]
target = 9
```

```python
def two_sum(nums, target):
    seen = {}                     # value -> index
    for i, x in enumerate(nums):
        need = target - x
        if need in seen:
            return [seen[need], i]
        seen[x] = i
    return []
# two_sum([2,7,11,15], 9) -> [0, 1]
```

Use this when: you need existence checks, counting, or "have I seen this before" — sorting would just waste `O(n log n)` for no benefit.

**Sample input for frequency counting**

```text
words = ["apple", "banana", "apple", "cherry", "apple", "banana"]
```

```python
from collections import Counter
freq = Counter(words)
# {'apple': 3, 'banana': 2, 'cherry': 1}
```

### 3.3 Option C — Heap / Priority Queue (top-k on unsorted data)

Avoids fully sorting when you only need the k best/worst elements.

**Sample input**

```text
nums = [7, 10, 4, 3, 20, 15]
k = 3
```

```python
import heapq

def top_k_largest(nums, k):
    return heapq.nlargest(k, nums)
# top_k_largest([7,10,4,3,20,15], 3) -> [20, 15, 10]
```

- Time: `O(n log k)` — much better than sorting all `n` when `k << n`
- Use this when: "top k", "kth largest", "k closest points", streaming max/min

### 3.4 Option D — Sliding Window / Prefix Sums (contiguous ranges in unsorted data)

Order in the array *is* meaningful here (it's not about sorted values, it's about position), so this works even though the values themselves are unsorted.

**Sample input**

```text
arr = [4, 2, 1, 7, 8, 1, 2, 8, 1, 0]
k = 3   # window size
```

```python
def max_sum_subarray(arr, k):
    window_sum = sum(arr[:k])
    best = window_sum
    for i in range(k, len(arr)):
        window_sum += arr[i] - arr[i - k]
        best = max(best, window_sum)
    return best
# max_sum_subarray(arr, 3) -> 17  (7+8+1... check window [7,8,1]=16, [8,1,2]=11 etc; adjust per data)
```

Use this when: the question is about a **contiguous** slice (longest/shortest/max-sum subarray or substring), regardless of whether values are sorted.

### 3.5 Option E — Streaming Data (can't hold it all / arrives over time)

**Sample input (conceptual stream)**

```text
stream: 5, 1, 9, 3, 7, 2, 8, ...  (arrives one at a time, unknown total length)
```

```python
import heapq

class RunningMedian:
    def __init__(self):
        self.small = []   # max-heap (negated)
        self.large = []   # min-heap

    def add(self, num):
        heapq.heappush(self.small, -num)
        heapq.heappush(self.large, -heapq.heappop(self.small))
        if len(self.large) > len(self.small):
            heapq.heappush(self.small, -heapq.heappop(self.large))

    def median(self):
        if len(self.small) > len(self.large):
            return -self.small[0]
        return (-self.small[0] + self.large[0]) / 2
```

Use this when: data isn't fully available upfront, so you can't sort once and be done — you need an incrementally-updated structure.

### 3.6 Option F — Graph/Relational Data (elements connected to each other)

Not about sorted vs unsorted at all — the input is inherently relational, so array-style patterns don't apply.

**Sample input**

```text
graph = {
    "A": ["B", "C"],
    "B": ["D"],
    "C": ["D"],
    "D": []
}
start = "A"
```

```python
from collections import deque

def bfs(graph, start):
    visited = {start}
    order = []
    q = deque([start])
    while q:
        node = q.popleft()
        order.append(node)
        for nxt in graph[node]:
            if nxt not in visited:
                visited.add(nxt)
                q.append(nxt)
    return order
# bfs(graph, "A") -> ['A', 'B', 'C', 'D']
```

Use this when: the input describes connections/dependencies (adjacency list/matrix, edges list) rather than a flat sequence.

---

## 4) Decision Table — Property → Pattern

| Input property | Exploit this pattern | Typical time |
|---|---|---|
| Sorted array | Binary search | `O(log n)` |
| Sorted array, need pair/triplet | Two pointers | `O(n)` |
| Unsorted, order doesn't matter, can sort | Sort + binary search/two pointers | `O(n log n)` |
| Unsorted, need existence/count, order irrelevant | Hash map/set | `O(n)` |
| Unsorted, need only top-k | Heap | `O(n log k)` |
| Unsorted, but position/contiguity matters | Sliding window / prefix sums | `O(n)` |
| Arrives incrementally, can't store all | Streaming structures (heaps, running stats) | `O(log k)` per item |
| Relational (graph/tree/dependency) | BFS / DFS / Union-Find | `O(V+E)` |
| Overlapping subproblems, optimal substructure | Dynamic programming | varies, often `O(n)`-`O(n^2)` |

---

## 5) Worked Comparison: Same Question, Sorted vs Unsorted Input

**Question:** Does any pair in the array sum to `target`?

**Case A — Input is sorted**

```text
arr = [1, 3, 4, 6, 9]
target = 10
```
→ Two pointers, `O(n)` time, `O(1)` space (no extra structure needed, because ordering already tells you which direction to move).

**Case B — Input is unsorted**

```text
arr = [9, 1, 6, 3, 4]
target = 10
```
→ Two options:
1. Sort first (`O(n log n)`), then two pointers (`O(n)`) → total `O(n log n)`
2. Hash set as you scan (`O(n)` time, `O(n)` space) — usually the better choice if you don't need the array sorted for anything else

```python
def has_pair_sum_unsorted(arr, target):
    seen = set()
    for x in arr:
        if target - x in seen:
            return True
        seen.add(x)
    return False
```

This is the key insight: **sortedness buys you `O(1)` extra space at the cost of `O(n log n)` to establish; a hash set buys you `O(n)` time immediately at the cost of `O(n)` extra space.** Choosing between them is a time/space trade-off once the "no ordering" property is confirmed.

---

## 6) Quick Checklist Before Coding

1. Is the input already sorted? → try binary search / two pointers first.
2. If unsorted, do I actually need it sorted for anything besides this one check? → if not, skip sorting, use a hash map/set.
3. Do I only need k elements, not all of them ordered? → heap, avoid full sort.
4. Is the question about a contiguous slice, not global order? → sliding window/prefix sums, sortedness is irrelevant here.
5. Is data arriving over time, not all at once? → streaming structure.
6. Are elements connected to each other rather than just a flat list? → graph traversal, not a search/sort pattern at all.

---

## Final Note

"Sorted vs unsorted" is really a proxy for a bigger question: **what property of this specific input can I exploit to avoid an O(n) or O(n²) brute-force scan?** Sorted order is one such property (enables binary search / two pointers). Frequency/existence needs point to hashing. Positional/contiguous constraints point to sliding windows. Relationships point to graphs. Always name the property explicitly before picking the pattern — that's what turns pattern selection from guesswork into a repeatable process.
