# 21 · Coding Problems and Patterns

LinkedIn's coding rounds skew toward **clean, applied** problems — not LeetCode-Hard tricks. The interviewer wants to see you think aloud, structure your code, handle edges, and discuss tradeoffs.

This file lists the patterns you'll see, the LinkedIn-reported problem set, Go-specific tips (since you write Go), and a discipline checklist.

## 21.1 The patterns LinkedIn really tests

In rough order of frequency from reported interview sets:

1. **Arrays / hashmaps** — counting, finding pairs, sliding window.
2. **Strings** — parsing, manipulation, sliding-window substring problems.
3. **Trees** — BFS / DFS, paths, lowest-common-ancestor, BST operations.
4. **Graphs** — BFS / DFS, topological sort, shortest path, connected components.
5. **Intervals** — merge intervals, find overlaps, scheduling.
6. **Linked lists** — manipulation, cycle detection, reversal.
7. **Heap / priority queue** — top-K, scheduling.
8. **Dynamic programming** — sometimes, but rarely contest-grade; typically 1D / 2D bottom-up.
9. **Backtracking** — combinations, permutations, constrained search.
10. **Stack / monotonic stack** — for problems with "next greater" kind of shape.
11. **Binary search** — both on arrays and on answer space (parametric search).
12. **Concurrent / OO design** — implement a thread-safe data structure, design a class.

## 21.2 Reported LinkedIn-specific problems

This is a curated list from public sources. Treat as *very likely to appear or close variants*.

### Trees / BST

- **Lowest Common Ancestor of a Binary Tree** (LC 236) and variants (BST, with parent pointer, of multiple nodes).
- **Diameter of Binary Tree** (LC 543).
- **Symmetric / mirror tree** (LC 101).
- **Maximum depth / max path sum** (LC 124).
- **Serialize and deserialize binary tree** (LC 297).
- **Closest BST value** (LC 270).
- **Validate BST** (LC 98).

### Strings / arrays

- **Sparse matrix multiplication** (LC 311) — a classic LinkedIn problem.
- **Repeated DNA sequences** (LC 187).
- **Find the celebrity** (LC 277).
- **Maximum subarray** (LC 53).
- **Two sum** variants (LC 1).
- **Longest substring without repeating characters** (LC 3).
- **Group anagrams** (LC 49).
- **Compare version numbers** (LC 165).
- **Read N characters given read4** (LC 157 / 158).
- **Wildcard matching** / **Regex matching** (LC 44 / 10).

### Graphs

- **Number of islands** (LC 200).
- **Word Ladder I / II** (LC 127 / 126).
- **Clone graph** (LC 133).
- **Course schedule** (LC 207 / 210) — topological sort.
- **Friend circles / number of provinces** (LC 547) — union-find.
- **Alien dictionary** (LC 269) — topological sort.

### Intervals / scheduling

- **Merge intervals** (LC 56).
- **Insert interval** (LC 57).
- **Meeting rooms** (LC 252 / 253).
- **Find missing intervals**.

### Heap / priority queue

- **Merge K sorted lists** (LC 23).
- **Kth largest element** (LC 215).
- **Top K frequent elements** (LC 347).
- **Find median from data stream** (LC 295).

### Design

- **LRU cache** (LC 146).
- **Design Twitter** (LC 355) — close to LinkedIn feed!
- **Design Tic Tac Toe** (LC 348).
- **Design HashMap / HashSet** (LC 706 / 705).
- **Design Add and Search Words Data Structure** (LC 211) — trie.

### Less common but reported

- **Maximum points on a line** (LC 149).
- **Evaluate reverse polish notation** (LC 150).
- **Factor combinations** (LC 254).
- **Paint house** (LC 256 / 265).
- **Shortest word distance** (LC 243 / 244 / 245).
- **Encode and decode strings** (LC 271).
- **Permutations II** (LC 47).

## 21.3 Interview discipline (must-do steps)

Even on a 20-min coding problem, do all of these:

1. **Restate the problem** in your words. Avoid misunderstandings.
2. **Clarify** with 1–2 targeted questions. *Bad*: "What about empty input?" (waste). *Good*: "Are the inputs sorted? Can they have duplicates? What's the size range — does an O(n²) solution work?"
3. **Brute-force first**, with complexity. Establish a baseline.
4. **Optimize**: 1–2 ideas. State the better complexity *before* coding.
5. **Choose data structures** consciously and announce them.
6. **Code** with named variables, early returns, no clever one-liners.
7. **Dry-run** with one realistic input and one edge case.
8. **Discuss tradeoffs** at the end: alternatives, scaling, follow-ups.

What separates Staff: between steps 4 and 5, talk through *why* you picked the optimization over alternatives. Mention the time/space tradeoff explicitly.

## 21.4 Go-specific tips for the interview

Given you write Go (per memory), here are LinkedIn-context tips:

- **Confirm Go is OK first.** Some interviewers prefer Java. Most accept Go; some don't. Ask.
- **Idiomatic Go is succinct**: `for k, v := range m { ... }`, `if err != nil { return err }`, short variable names in tight scopes.
- **Use the right primitives**:
  - `map[KeyType]ValueType` for hashmap.
  - `[]T` for dynamic array.
  - `container/heap` for priority queue (boilerplate; practice).
  - `container/list` for doubly-linked list (rarely needed; just use slices).
- **Concurrency**: goroutines + channels for problems that ask for concurrent solutions.
  - `sync.WaitGroup` for "wait for N goroutines".
  - `sync.Mutex` for shared state.
  - `context.Context` for cancellation.
- **Avoid pitfalls**:
  - Closure over loop variable (Go 1.22 fixed this; older versions: assign to local).
  - Slice append realloc — `cap()` matters when sharing slices.
  - `nil` map writes panic — always `make(map)` first.

### Go heap boilerplate (memorize this)

```go
import "container/heap"

type IntHeap []int

func (h IntHeap) Len() int           { return len(h) }
func (h IntHeap) Less(i, j int) bool { return h[i] < h[j] } // min-heap
func (h IntHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }

func (h *IntHeap) Push(x any) {
    *h = append(*h, x.(int))
}

func (h *IntHeap) Pop() any {
    old := *h
    n := len(old)
    x := old[n-1]
    *h = old[:n-1]
    return x
}

// usage
h := &IntHeap{}
heap.Init(h)
heap.Push(h, 5)
heap.Push(h, 1)
top := heap.Pop(h).(int) // 1
```

### Go LRU cache template

```go
type LRUNode struct {
    key, val   int
    prev, next *LRUNode
}

type LRUCache struct {
    cap        int
    m          map[int]*LRUNode
    head, tail *LRUNode // sentinels
}

func New(cap int) *LRUCache {
    h, t := &LRUNode{}, &LRUNode{}
    h.next, t.prev = t, h
    return &LRUCache{cap: cap, m: map[int]*LRUNode{}, head: h, tail: t}
}

func (c *LRUCache) Get(key int) int {
    n, ok := c.m[key]
    if !ok { return -1 }
    c.detach(n); c.attachFront(n)
    return n.val
}

func (c *LRUCache) Put(key, val int) {
    if n, ok := c.m[key]; ok {
        n.val = val; c.detach(n); c.attachFront(n); return
    }
    if len(c.m) == c.cap {
        lru := c.tail.prev
        c.detach(lru); delete(c.m, lru.key)
    }
    n := &LRUNode{key: key, val: val}
    c.attachFront(n); c.m[key] = n
}

func (c *LRUCache) detach(n *LRUNode) {
    n.prev.next, n.next.prev = n.next, n.prev
}

func (c *LRUCache) attachFront(n *LRUNode) {
    n.next = c.head.next; n.prev = c.head
    c.head.next.prev = n; c.head.next = n
}
```

You should be able to type this out from memory in <5 minutes.

## 21.5 Concurrency problem templates (Go)

LinkedIn loves "make this concurrent" follow-ups. Two templates to memorize:

### Worker pool

```go
func workerPool(tasks []Task, n int) []Result {
    in := make(chan Task)
    out := make(chan Result, len(tasks))
    var wg sync.WaitGroup

    for i := 0; i < n; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for t := range in {
                out <- process(t)
            }
        }()
    }

    go func() {
        for _, t := range tasks { in <- t }
        close(in)
    }()

    go func() {
        wg.Wait()
        close(out)
    }()

    results := make([]Result, 0, len(tasks))
    for r := range out { results = append(results, r) }
    return results
}
```

### Cancellable context pattern

```go
func searchWithTimeout(ctx context.Context, query string) (Result, error) {
    ctx, cancel := context.WithTimeout(ctx, 200*time.Millisecond)
    defer cancel()

    resultCh := make(chan Result, 1)
    errCh := make(chan error, 1)

    go func() {
        r, err := doSearch(ctx, query)
        if err != nil { errCh <- err; return }
        resultCh <- r
    }()

    select {
    case r := <-resultCh: return r, nil
    case err := <-errCh: return Result{}, err
    case <-ctx.Done(): return Result{}, ctx.Err()
    }
}
```

## 21.6 Going from problem statement to clean code

A useful self-rubric: after you've coded, ask yourself:

1. **Did I name variables descriptively?** (`leftPtr` not `lp`.)
2. **Did I extract a helper for any 5+-line repeated block?**
3. **Did I handle nil / empty input?**
4. **Did I avoid mutating arguments unless required?**
5. **Did I avoid hidden side effects?**
6. **Could a reviewer understand this in 30 seconds without my explanation?**

## 21.7 What to ask the interviewer when stuck

- "I'm thinking of approach X but it's O(n²). Is there a constraint I can exploit?"
- "Let me trace through a small example — would you mind if I write it out?"
- "Is the input guaranteed to be valid here, or should I validate?"

Asking is fine; staying silent for 4 minutes is fatal.

## 21.8 Tradeoffs and follow-ups to anticipate

Frequent extension questions on coding rounds:

- **"Now make it concurrent / multi-threaded."**
- **"Now make it streaming — input arrives one at a time."**
- **"Now it's distributed across N machines."**
- **"Now optimize memory — we have 1GB but the input is 100GB."**
- **"What if the input is sorted?"**
- **"What if duplicates aren't allowed?"**
- **"What's the most memory-efficient version?"**

Have a habit of preempting these: "I've optimized for time here; if memory was the constraint, I'd use approach Y."

## 21.9 The "I don't know this problem" recovery

If a problem stumps you cold:
- State that openly: "I haven't seen this exact problem; let me think through it from first principles."
- Start with brute force and a small example.
- Articulate why your brute force is too slow.
- Propose two candidate optimizations; pick one and start.
- If you fail, the interviewer often offers a hint — accept it gracefully, integrate, and continue.

A staff candidate doesn't pretend to know things they don't; they show how they think under uncertainty.

## 21.10 One more pattern: streaming median (LC 295)

LinkedIn-favorite "design a class" question. Solution:

- Two heaps: max-heap of lower half, min-heap of upper half.
- Maintain `|low| - |high| ∈ {0, 1}`.
- On `addNum(x)`:
  - If x < low.top → push to low.
  - Else → push to high.
  - Rebalance.
- `findMedian()`:
  - If `|low| > |high|` → low.top.
  - Else → (low.top + high.top) / 2.

Both ops `O(log n)`. Memorize.