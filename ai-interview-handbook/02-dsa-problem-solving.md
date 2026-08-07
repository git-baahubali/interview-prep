# DSA & Problem Solving

Patterns that actually appear in AI/ML engineering screens: hashing, two pointers, sliding windows, heaps, and graph traversal. Every answer states complexity **and why**.

**Questions:** 8

---

## Easy

---

## Q1: Two Sum — find two indices whose values add to a target.

### Answer

Store each value you have already seen in a hash map from value → index. For each new value, check whether its complement is already in the map. One pass, no sorting, and it returns the original indices.

### Example

```python
def two_sum(nums, target):
    seen = {}

    for i, value in enumerate(nums):
        needed = target - value

        if needed in seen:
            return [seen[needed], i]

        seen[value] = i

    return []
```

**Time Complexity:** O(n) — one pass over `nums`, and each dict lookup/insert is O(1) average.

**Space Complexity:** O(n) — worst case the map holds every element before a match is found.

**Why not sort?** Sorting plus two pointers is O(n log n) and destroys the original indices, so you would need to track them separately. Hashing is strictly better here.

### Interview Follow-ups

- What if the array is already sorted? (Two pointers, O(n) time and O(1) space.)
- What if you must return **all** pairs? (Watch out for duplicates; count occurrences instead of storing a single index.)
- Can the same element be used twice? Clarify this before coding. (Normally no — and the single-pass hash map handles it for free, because you look up the complement *before* inserting the current element, so an element can never pair with itself. If reuse were allowed, `target = 2 * nums[i]` would be a valid answer and you would check the map after inserting instead.)

---

## Q2: Reverse a linked list.

### Answer

Walk the list once, re-pointing each node's `next` to the previous node. You need three pointers: `prev`, `curr`, and a saved `next` so you do not lose the rest of the list.

### Example

```python
def reverse_list(head):
    prev = None
    curr = head

    while curr:
        nxt = curr.next     # save the rest
        curr.next = prev    # re-point backwards
        prev = curr         # advance
        curr = nxt

    return prev             # new head
```

**Time Complexity:** O(n) — each node is visited exactly once.

**Space Complexity:** O(1) — only three pointers, regardless of list length. (The recursive version is O(n) space because of the call stack.)

### Interview Follow-ups

- Write it recursively, and state why that is O(n) space. (Recurse to the tail, then rewire on the way back: `def rev(node): if not node or not node.next: return node; head = rev(node.next); node.next.next = node; node.next = None; return head`. It is O(n) space because the call stack holds one frame per node until the base case is reached — and on a long list it will hit Python's recursion limit, which is exactly why the iterative three-pointer version is the production answer.)
- How would you reverse only nodes between positions `m` and `n`? (Walk to node `m-1` and hold it as `prev_tail`, remember `start = prev_tail.next` (which becomes the tail of the reversed segment), reverse exactly `n - m + 1` nodes with the standard three-pointer loop, then reconnect: `prev_tail.next = new_head` and `start.next = node_after_n`. Use a dummy head node so `m == 1` needs no special case. Still O(n) time, O(1) space — the trick is holding the two boundary pointers before you start mutating.)
- How do you detect a cycle before reversing? (Floyd's tortoise and hare.)

---

## Q3: Check whether a string is a valid palindrome, ignoring non-alphanumeric characters and case.

### Answer

Two pointers from both ends, skipping characters that are not alphanumeric, comparing lowercased characters. This avoids building a cleaned copy of the string.

### Example

```python
def is_palindrome(s):
    left, right = 0, len(s) - 1

    while left < right:
        while left < right and not s[left].isalnum():
            left += 1
        while left < right and not s[right].isalnum():
            right -= 1

        if s[left].lower() != s[right].lower():
            return False

        left += 1
        right -= 1

    return True
```

**Time Complexity:** O(n) — each pointer only moves forward/backward, so total movement is bounded by n.

**Space Complexity:** O(1) — no extra structure. The one-liner `s = [c.lower() for c in s if c.isalnum()]` is also O(n) time but O(n) space.

### Interview Follow-ups

- How would you handle Unicode (accents, casefolding)? (`str.casefold()`, `unicodedata.normalize`.)
- What changes if you are allowed to delete at most one character? (Run the same two-pointer scan; on the first mismatch you have exactly two candidate repairs — skip the left character or skip the right one — so return `is_pal(s, l+1, r) or is_pal(s, l, r-1)` using a plain helper that checks a plain palindrome with no further deletions. Still O(n) time, because each helper call is a single linear scan and you only ever branch once, and O(1) space. The insight interviewers want: you do not need DP, and you must not allow a second deletion inside the helper.)

---

## Intermediate

---

## Q4: Longest substring without repeating characters.

### Answer

Sliding window with a map from character → last index seen. Expand the right edge; when you hit a duplicate that lies **inside** the current window, jump the left edge to just past its previous occurrence.

### Example

```python
def length_of_longest_substring(s):
    last_seen = {}
    best = 0
    left = 0

    for right, ch in enumerate(s):
        if ch in last_seen and last_seen[ch] >= left:
            left = last_seen[ch] + 1     # shrink window past the duplicate

        last_seen[ch] = right
        best = max(best, right - left + 1)

    return best
```

**Time Complexity:** O(n) — `right` advances n times and `left` only ever moves forward, so the total pointer movement is O(n). This is why the window is not O(n²) despite the nested logic.

**Space Complexity:** O(min(n, k)) where k is the alphabet size — the map holds at most one entry per distinct character.

**The key check** is `last_seen[ch] >= left`. Without it, a duplicate from *before* the window would incorrectly drag `left` backwards.

### Interview Follow-ups

- Generalise to "at most K distinct characters." (Same sliding window, but the state becomes a `Counter` of characters in the window instead of a last-seen index map. Expand `right` and increment the count; while `len(counter) > K`, decrement `counter[s[left]]`, delete the key when it hits zero, and advance `left`. Record the best width each iteration. O(n) time — each pointer moves forward at most n times — and O(K) space. The original problem is just this with K unbounded and duplicates disallowed.)
- This pattern shows up in token-budget windows for LLM context — how would you adapt it to a max token count instead of max distinct chars? (Replace the distinct-character constraint with a running sum: keep `total` as the token count of messages in the window, and while `total > budget`, subtract `tokens[left]` and advance `left`. That gives the longest suffix of conversation history that fits the budget in O(n). Two practical caveats that make it a real problem rather than an exercise: token counts are not additive across a boundary if you re-render a template, so measure the assembled prompt rather than summing parts; and you usually want the window anchored at the *right* end (most recent turns) with the system prompt pinned outside it, which turns it into a one-directional shrink rather than a search for the maximum window.)

---

## Q5: Top K frequent elements.

### Answer

Count frequencies with a hash map, then select the K largest counts with a **min-heap of size K**. Push each item; when the heap exceeds K, pop the smallest. The heap always holds the current best K.

### Example

```python
import heapq
from collections import Counter

def top_k_frequent(nums, k):
    counts = Counter(nums)
    heap = []                                  # min-heap of (count, value)

    for value, count in counts.items():
        heapq.heappush(heap, (count, value))
        if len(heap) > k:
            heapq.heappop(heap)                # drop the least frequent

    return [value for count, value in heap]
```

**Time Complexity:** O(n + m log k) where n = len(nums) and m = number of distinct values. Counting is O(n); each of the m items costs O(log k) heap work. Since k ≤ m, this beats the O(m log m) of a full sort when k is small.

**Space Complexity:** O(m) for the counter plus O(k) for the heap.

**Alternatives:**

| Approach | Time | When to use |
|---|---|---|
| Sort all counts | O(m log m) | Simplest; fine when m is small |
| Min-heap of size k | O(n + m log k) | k ≪ m, the usual interview answer |
| Bucket sort by count | O(n) | Counts are bounded by n — buckets `0..n` |
| `Counter.most_common(k)` | O(m log k) internally | Production code; say you know what it does |

### Interview Follow-ups

- Solve it in true O(n) with bucket sort. (Counts are bounded by `n`, so allocate `buckets = [[] for _ in range(n+1)]`, append each element to `buckets[count]`, then walk the buckets from index `n` down to `1` collecting elements until you have k. That is O(n) time and O(n) space, beating the heap's O(n log k) — the trick is recognising that the *values being sorted* have a bounded integer range, which is the precondition for any counting/bucket sort. In practice the heap version is what you write unless the interviewer explicitly asks for linear time, since it uses less memory and k is usually small.)
- How would you do this over a **stream** where you cannot store all items? (Count-Min Sketch / Space-Saving — a genuinely relevant answer for telemetry on an LLM gateway.)

---

## Q6: Detect a cycle in a directed graph.

### Answer

DFS with three states per node: unvisited, **in the current recursion stack**, and fully done. If DFS reaches a node that is currently in the recursion stack, that is a back edge, which means a cycle. Merely "already visited" is not enough — a node can be revisited legitimately via a different path in a DAG.

### Example

```python
WHITE, GREY, BLACK = 0, 1, 2

def has_cycle(graph):
    state = {node: WHITE for node in graph}

    def dfs(node):
        state[node] = GREY                    # on the current path
        for neighbour in graph.get(node, []):
            if state.get(neighbour, WHITE) == GREY:
                return True                   # back edge -> cycle
            if state.get(neighbour, WHITE) == WHITE and dfs(neighbour):
                return True
        state[node] = BLACK                   # fully explored
        return False

    return any(dfs(n) for n in graph if state[n] == WHITE)
```

**Time Complexity:** O(V + E) — every vertex is coloured once and every edge inspected once.

**Space Complexity:** O(V) for the state map plus O(V) recursion depth worst case.

**Why this matters in AI engineering:** this is exactly the validation an agent framework runs on a workflow graph. A LangGraph-style graph *permits* cycles by design, so instead of rejecting them you bound them with a recursion/step limit — but you still need cycle detection to distinguish an intentional loop from an accidental one, and topological order for parallel scheduling.

### Interview Follow-ups

- How do you produce a topological order? (Kahn's algorithm, or reverse DFS post-order — and it fails exactly when a cycle exists.)
- How does undirected cycle detection differ? (Union-Find, or DFS ignoring the immediate parent.)

---

## Advanced

---

## Q7: Merge K sorted lists (or K sorted retrieval result sets).

### Answer

Keep a min-heap holding the current head of each list. Pop the global minimum, append it to the output, and push the next element from the list it came from. The heap never exceeds K entries.

### Example

```python
import heapq

def merge_k_sorted(lists):
    heap = []
    for list_idx, lst in enumerate(lists):
        if lst:
            # (value, which list, index in that list)
            heapq.heappush(heap, (lst[0], list_idx, 0))

    out = []
    while heap:
        value, list_idx, elem_idx = heapq.heappop(heap)
        out.append(value)

        nxt = elem_idx + 1
        if nxt < len(lists[list_idx]):
            heapq.heappush(heap, (lists[list_idx][nxt], list_idx, nxt))

    return out
```

**Time Complexity:** O(N log K) where N is the total number of elements across all lists. Every element is pushed and popped once, each at O(log K) because the heap holds at most K items.

**Space Complexity:** O(K) for the heap, plus O(N) for the output.

**Why the tuple includes indices:** it breaks ties deterministically and tells you which list to refill. If values could be non-comparable objects, tie-breaking on `list_idx` prevents a `TypeError`.

**Production parallel:** this is how you merge results from sharded vector index partitions, or from several retrievers, when each shard already returns results sorted by score. Naively concatenating and sorting is O(N log N); the heap merge is strictly better and can stop early once you have the top-k.

### Interview Follow-ups

- How do you stop after the global top-k without draining every list? (Break out once `len(out) == k` — total cost O(k log K).)
- How does this change if scores come from different retrievers on incomparable scales? (Normalise, or use Reciprocal Rank Fusion — see the RAG file.)

---

## Q8: How do you approach an unfamiliar coding problem in an interview?

### Answer

A repeatable process matters more than pattern recall, because interviewers score your communication as much as your solution.

1. **Restate and clarify.** Input types, size bounds, duplicates, empty input, sorted or not, return value vs in-place. Say the constraints out loud — bounds hint at the target complexity (n ≤ 20 suggests exponential/bitmask; n ≤ 10⁵ suggests O(n log n)).
2. **Walk one small example by hand.** Then a tricky one (empty, single element, all-duplicates).
3. **State a brute force and its complexity.** This guarantees you have *something* and anchors the improvement.
4. **Name the bottleneck, then map it to a pattern.** "I am re-scanning the prefix" → hash map or prefix sums. "I need the k best" → heap. "Sorted input" → two pointers / binary search. "Overlapping subproblems" → DP.
5. **State the target complexity before coding**, and get agreement.
6. **Code cleanly**: meaningful names, guard clauses first, no premature micro-optimisation.
7. **Dry-run your own code** on the small example, then the edge cases.
8. **State final time and space complexity and justify each.**

**Pattern → signal cheat sheet:**

| Signal in the problem | Likely pattern |
|---|---|
| "Pair/triplet summing to X" | Hash map, or sort + two pointers |
| "Contiguous subarray/substring" | Sliding window, prefix sums |
| "Sorted array, find position" | Binary search |
| "K largest / smallest / closest" | Heap, or quickselect |
| "Number of ways / min cost" | Dynamic programming |
| "Connected / reachable / dependencies" | BFS, DFS, Union-Find, topological sort |
| "Next greater / balanced brackets" | Stack |
| "Prefix / autocomplete" | Trie |
| "Intervals overlapping" | Sort by start, sweep line |

**Common misconception:** the goal is not to reach the optimal solution instantly. A clearly communicated brute force that you then improve scores better than silence followed by a half-written optimal answer.

### Interview Follow-ups

- What do you do when you are genuinely stuck? (Say what you have tried, state the invariant you want, and ask for a nudge — this is a data point in your favour, not against you.)

---
