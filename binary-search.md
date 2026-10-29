# Binary search

Binary search applies to sorted data or to a monotonic condition. Each iteration cuts the remaining search range roughly in half, so it takes `O(log n)` time.

## Common loop shapes

### Exact match: `left <= right`

Use this when you need any index containing `target`. The range is inclusive, and the loop can return as soon as it finds a match.

```java
int left = 0, right = nums.length - 1;
while (left <= right) {
    int mid = left + (right - left) / 2;
    if (nums[mid] == target) return mid;
    if (nums[mid] < target) left = mid + 1;
    else right = mid - 1;
}
return -1;
```

### Lower bound and upper bound: `left < right`

These are boundary searches over a sorted array, using the half-open range `[left, right)` (the left endpoint is included; the right endpoint is excluded):

- **Lower bound** is the first index whose value is **greater than or equal to** `target` (`nums[i] >= target`). If no value meets that condition, it is `nums.length`.
- **Upper bound** is the first index whose value is **strictly greater than** `target` (`nums[i] > target`). If no value meets that condition, it is `nums.length`.

- Both return insertion positions. Lower bound inserts before existing equal values; upper bound inserts after them.

**Remember:**

- Lower bound uses `< target` to move `left` right.
- Upper bound uses `<= target` to move `left` right.
- The difference `upperBound - lowerBound` is the number of occurrences of `target`.
- The first occurrence is the lower bound; the last occurrence is one before the upper bound. 

For lower bound:

```java
int left = 0, right = nums.length;
while (left < right) {
    int mid = left + (right - left) / 2;
    if (nums[mid] < target) left = mid + 1;
    else right = mid;
}
// left is the first index with nums[left] >= target, or nums.length
return left;
```

Example: for `[1, 3, 3, 5, 8]`, lower bound of `3` is index `1`; lower bound of `4` is index `3`, its insertion position; lower bound of `9` is `5` (`nums.length`).

For upper bound, use this condition inside the same loop:

```java
if (nums[mid] <= target) left = mid + 1;
else right = mid;
```

In the same example, upper bound of `3` is index `3`, just after the two `3`s. 

#### Why set `right = mid` instead of `mid - 1`?

When the condition says `nums[mid]` is at or beyond the boundary, `mid` may itself be the first qualifying index. Setting `right = mid` keeps that possible answer in the search range while narrowing the range. Setting `right = mid - 1` would discard it. This works with the half-open range `[left, right)`; the excluded endpoint is `right`, not `mid`.

**Remember:** in a boundary search, keep `mid` when it could be the answer (`right = mid`). Move past it only when it cannot be the answer (`left = mid + 1`).

By contrast, exact-match search above uses the inclusive range `[left, right]`. There, when `nums[mid]` is not the target, it can safely discard `mid` with `right = mid - 1` (or `left = mid + 1`).

### Neighbor search: `left + 1 < right`

Use this when the final answer depends on checking both neighboring candidates. The loop stops with the two candidates `left` and `right`; check them after the loop, including bounds.

```java
int left = 0, right = nums.length - 1;
while (left + 1 < right) {
    int mid = left + (right - left) / 2;
    if (nums[mid] < target) left = mid;
    else right = mid;
}
// Check left and right as appropriate for the problem.
```

## Common applications

| Task | Approach |
| --- | --- |
| Find a value in a sorted array | Exact-match search |
| Find first occurrence / insertion position | Lower bound |
| Find last occurrence / count duplicates | Upper bound, often paired with lower bound |
| Find a transition in a monotonic predicate | Boundary search for first true or last false |
| Find a minimum feasible capacity, speed, or time | Binary search the answer; test each candidate with a feasibility function |
| Search a rotated sorted array | Identify the sorted half at each step, then keep the half that can contain the target |
| Find a minimum in a rotated array | Compare the middle value with the right endpoint to discard a sorted portion |

For rotated arrays with duplicates, when `nums[left] == nums[mid] == nums[right]`, the sorted half is ambiguous; shrink both ends (`left++`, `right--`).

## References

- [Binary search for beginners](https://leetcode.com/discuss/general-discussion/691825/binary-search-for-beginners-problems-patterns-sample-solutions)
- [Lower and upper bound notes](https://medium.com/swlh/binary-search-find-upper-and-lower-bound-3f07867d81fb)
