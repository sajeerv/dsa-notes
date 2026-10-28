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

### Boundary search: `left < right`

Use a half-open range `[left, right)` to find an insertion position, such as the first element greater than or equal to a target. This is the lower-bound pattern.

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

To find the upper bound—the first index with a value greater than `target`—change the condition to `nums[mid] <= target`:

```java
if (nums[mid] <= target) left = mid + 1;
else right = mid;
```

Lower and upper bounds are useful for duplicates: the first occurrence is the lower bound, the last occurrence is one before the upper bound, and their difference gives the number of occurrences.

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
