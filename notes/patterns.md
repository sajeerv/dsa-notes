# Patterns

Short reminders. Expand a section when a problem teaches you something durable.

## Arrays / hashing

- Need O(n) lookups? Put values (or complements) in a `HashMap` / `HashSet`.
- Subarray sums: prefix sum + map of `prefix -> index or count`.

## Two pointers

- Sorted array pair/triplet search.
- In-place overwrite from the left while scanning from the left or both ends.

## Sliding window

- Grow `right`, shrink `left` while the window is invalid.
- Track a count, sum, or frequency map of the window.

## Binary search

- Search the **answer space**, not only a sorted array.
- Loop as `lo < hi` with a clear invariant: what does `lo` mean after the loop?
- Three loop shapes (`left <= right`, `left < right`, `left + 1 < right`) are in [`java-dsa-cheatsheet.md`](java-dsa-cheatsheet.md).

## Linked list

- Dummy head for insert/delete at the front.
- Fast/slow for middle, cycle, k-th from end.

## Stack

- Monotonic stack: next greater / next smaller in O(n).
- Parse nested structure (parentheses, paths).

## Trees

- Recursion: left/right results + current node.
- BFS with a queue for level order.

## Graphs

- Adjacency list from edge list.
- BFS for shortest unweighted path; DFS/recursion or stack for components.
- Kahn’s algorithm or DFS coloring for cycle / topological order.

## DP

- Define `dp[i]` (or `dp[i][j]`) in one sentence before coding.
- Bottom-up when the recurrence and bounds are obvious; memo DFS when the state is sparse.

## Backtracking

- Choose → recurse → un-choose.
- Prune early; sort to skip duplicates.

## Complexity

Always write time and extra space in `notes.md`. Count the cost of sorting, maps, and recursion depth.
