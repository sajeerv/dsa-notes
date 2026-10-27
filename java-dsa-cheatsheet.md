# Java DSA cheat sheet

The rest of the old Pages “Java Cheat Sheets” file: language tricks, regex, graphs, DP, binary search. Collections Big-O lives in [`java-collections.md`](java-collections.md).

## Arrays and strings

```java
int[] b = a.clone();
System.arraycopy(src, srcPos, dest, destPos, len);
int[] copy = Arrays.copyOf(orig, newLen);           // truncate or pad with 0
int[] range = Arrays.copyOfRange(arr, 2, 6);        // toIndex exclusive
Arrays.fill(result, -1);
Arrays.toString(a);
return new int[] { 1, 3 };

Character.isLetter(chars[i]);
Character.toLowerCase(chars[i]);
Character.toUpperCase(chars[i]);
Character.getNumericValue(ch);

sb.setLength(n);   // drop a suffix; useful in backtracking
```

Integer division loses the fraction unless you force a double:

```java
double avgWrong = sum / 10;
double avg = sum * 1.0 / 10;
```

Digit to char (not ASCII decode):

```java
int a = 1;
char c = (char) (a + '0');
```

Split / replace:

```java
String[] words = str.split("\\s+");     // one or more whitespace
s.split("[()]");                        // split on ( or )
"a/b/c".split("/");                     // [a, b, c]
"/a/b/c".split("/");                    // ["", a, b, c]  leading empty
p.replaceAll("\\W+", " ");              // non-word -> space
"a:b:c".split(":");
```

Regex metacharacters that must be escaped if you mean the literal: `<([{\\^-=$!|]})?*+.>`  
In a Java string the backslash itself is `\\\\`. Limit-`n` splits if you do not want empty tokens from repeated delimiters.

Replace from the **end** when indexes would otherwise shift.

## Regex reference

### Character classes

| Pattern | Meaning |
| --- | --- |
| `[abc]` | a, b, or c |
| `[^abc]` | not a, b, or c |
| `[a-zA-Z]` | range |
| `[a-d[m-p]]` | union: a–d or m–p |
| `[a-z&&[def]]` | intersection: d, e, or f |
| `[a-z&&[^bc]]` | subtraction: a–z except b, c |
| `[a-z&&[^m-p]]` | a–z except m–p |

### Predefined classes

| Pattern | Meaning |
| --- | --- |
| `.` | any character |
| `\d` | digit `[0-9]` |
| `\D` | non-digit |
| `\s` | whitespace `[ \t\n\x0B\f\r]` |
| `\S` | non-whitespace |
| `\w` | word `[a-zA-Z_0-9]` |
| `\W` | non-word |
| `\t` `\n` `\r` `\f` | tab, newline, CR, form feed |
| `\\` | literal backslash |

### Quantifiers and grouping

| Pattern | Meaning |
| --- | --- |
| `X?` | once or not at all |
| `X*` | zero or more |
| `X+` | one or more |
| `X{n}` | exactly n |
| `X{n,}` | at least n |
| `X{n,m}` | n to m |
| `X\|Y` | X or Y |
| `(X)` | capturing group |
| `XY` | X followed by Y |

### String API

| Method | Meaning |
| --- | --- |
| `s.matches(regex)` | true only if the **whole** string matches |
| `s.split(regex)` | pieces; the delimiter is dropped |
| `s.replaceFirst(regex, repl)` | first match |
| `s.replaceAll(regex, repl)` | all matches |

## Matrix helpers

```java
int[][] dirs = {
    {0, 0}, {0, -1}, {0, 1},
    {-1, 0}, {-1, 1}, {-1, -1},
    {1, 0}, {1, -1}, {1, 1}
};
for (int[] d : dirs) {
    int i = r + d[0], j = c + d[1];
}
```

- Same main diagonal: constant `i - j`
- Same anti-diagonal: constant `i + j`
- Flat index `0 .. M*N - 1` with `N` columns: `row = index / N`, `col = index % N`

Circular arrays: duplicate the array (second half = first half) when that simplifies wrap-around.

## Numbers, bits, overflow

```java
// overflow before appending a digit
if (result > Integer.MAX_VALUE / 10
        || (result == Integer.MAX_VALUE / 10 && ch - '0' > 7)) {
    // overflow
}

boolean differentSigns = (a < 0) != (b < 0);

a = -a;          // not a * -1 (overflow at Integer.MIN_VALUE)
a >> 1;          // divide by 2 (watch sign)
a << 1;          // * 2
a += a;

Integer.highestOneBit(n);
Integer.lowestOneBit(n);
```

Java `%`: sign follows the **dividend**. `x % y` equals `x % -y`.

Factors of `n`: loop `i` to `sqrt(n)`, add `i` and `n/i`, skip the duplicate when they are equal.

Unique “contains all parts” hash: product of primes, `mod` each multiply to avoid overflow. `a` contains `b`’s parts if `hash(a) % hash(b) == 0` (with care around collisions).

Interval overlap of `[a0, a1]` and `[b0, b1]`: `(max(a0, b0), min(a1, b1))`.

If min/max sentinels are unclear, use `Integer` and a null check instead of `Integer.MAX_VALUE`.

Hash a grid point: `x * width + y`.

### Combinatorics

- Permutations with repetition, order matters: `n^r`
- Permutations no repetition: `n! / (n - r)!`
- Combinations no repetition: `n! / ((n - r)! r!)`

### Distance / sequences

- Euclidean: `sqrt((x1-x2)^2 + (y1-y2)^2)`
- Manhattan: `|x1-x2| + |y1-y2|`
- AP nth term: `a1 + (n-1)d`
- AP sum: `n * (a1 + an) / 2` or `n * (2*a1 + (n-1)d) / 2`

### randM from randN

Scale to a product, reject the remainder, then map down. Examples:

- `rand10` from `rand7`: `rand7 → rand49 → rand40 → rand10`
- `rand11` from `rand3`: `rand3 → rand27 → rand22 → rand11`
- `rand9` from `rand7`: `rand7 → rand49 → rand45 → rand9`
- `rand13` from `rand6`: `rand6 → rand36 → rand26 → rand13`

## Heap / recursion habits

If a helper wants arguments in a fixed order:

```java
int divide(int a, int b) {
    if (a < b) return divide(b, a);
    ...
}
```

You can return a field or array slot directly: `return helper(root)[2];`

## Algorithm templates (short)

- Backtracking: permutations, subset sum
- Prefer `ArrayDeque` for stacks/queues

## Graphs

Classify: directed vs undirected, weighted vs unweighted, cyclic vs acyclic. Directed acyclic → DAG. Undirected acyclic connected → tree.

**Back edge:** to self or to an ancestor in the DFS tree.

### BFS

Queue. Vertices reachable from a source. Loop over unvisited vertices for a disconnected graph.

Uses: unweighted shortest path (fewest edges), connected components (one BFS per unused vertex).

### DFS

Stack / recursion. Coloring, bipartite check (two colors; neighbor must differ), chromatic number (NP-hard in general).

### Cycle — undirected

1. DFS with parent: visited neighbor that is not parent → cycle.
2. BFS with parent (or drop the reverse edge from the neighbor list).
3. Union-find: edge `(u, v)` whose ends are already in the same set → cycle.

### Cycle — directed

1. Extra `visiting[]` (on the recursion stack). Seeing `visiting[v] == true` → back edge.
2. Colors: WHITE never seen, GRAY in progress, BLACK done. GRAY neighbor → cycle.
3. Topological sort that does not emit every vertex → cycle.

### Topological sort

1. DFS: when a vertex finishes, add it at the **front** of a list.
2. Kahn (BFS): enqueue indegree 0, emit, decrement neighbors; leftover indegree > 0 means a cycle.

### Shortest / longest path on a DAG

Topological order, then relax edges.

- Shortest: `dist[] = INF`, `dist[s] = 0`, if `dist[v] > dist[u] + w(u,v)` update.
- Longest: `dist[] = -INF`, `dist[s] = 0`, if `dist[v] < dist[u] + w(u,v)` update.

General weighted graphs: Dijkstra / Bellman-Ford, not this DAG trick.

## Dynamic programming

**Unbounded coin change (min coins)** for amount `S`:

```
minCoins[amount] = min(1 + minCoins[amount - c]) for c in coins
```

Loop amount outer, coin inner. Limited supply: do not use coin `c` if that amount already used `max(c)` copies.

If denominations are fixed (1, 10, 25), greedy by largest coin is often enough.

**Unbounded coin change (number of ways):** coin outer, amount inner (`amount >= coin`).

**0/1 knapsack:** `dp[i][j]` = best value using first `i` items with capacity `j`. Skip item `i` if `weight[i] > j`; else `max(skip, value[i] + dp[i-1][j - weight[i]])`.

## Binary search

Three loop shapes (LeetCode templates):

**Template 1 — `left <= right`**  
Standard find. Condition does not need neighbors. Loop can return immediately. Empty range ⇒ not found.

```java
int left = 0, right = n - 1;
while (left <= right) {
    int mid = left + (right - left) / 2;
    if (nums[mid] == target) return mid;
    if (nums[mid] < target) left = mid + 1;
    else right = mid - 1;
}
// right + 1 == left; no candidate
```

**Template 2 — `left < right`**  
Need the element to the right of `mid`. Search space stays size ≥ 2. One candidate left; post-process.

```java
int left = 0, right = n;
while (left < right) {
    int mid = left + (right - left) / 2;
    if (nums[mid] < target) left = mid + 1;
    else right = mid;
}
```

**Template 3 — `left + 1 < right`**  
Need both neighbors. Two candidates left; post-process.

```java
int left = 0, right = n - 1;
while (left + 1 < right) {
    int mid = left + (right - left) / 2;
    if (nums[mid] < target) left = mid;
    else right = mid;
}
```

| Task | Idea |
| --- | --- |
| Insert position | Standard search; return `left` |
| Closest value | After search, compare `left` / `right` (bounds check) |
| Rotation index | If `mid > mid+1`, return `mid+1` |
| Rotated search | If left..mid sorted, see if target is there; else the right half |
| Rotated + duplicates | If `left == mid == right`, shrink `left++` / `right--` (`[3,1,2,3,3,3,3]`) |
| Min in rotated (no dups) | If `left < right` no rotation; else drop the sorted half |

Lower / upper bound notes: [Binary search lower and upper bound](https://medium.com/swlh/binary-search-find-upper-and-lower-bound-3f07867d81fb)

## Sorting reminder

Bubble sort: adjacent swaps, largest sinks to the end. Stop early if a pass has no swaps; inner loop can ignore the sorted tail.

## Links (from the original notes)

- [Useful LeetCode links](https://leetcode.com/discuss/general-discussion/665604/important-and-useful-links-from-all-over-the-leetcode)
- [Bit manipulation summary](https://leetcode.com/problems/sum-of-two-integers/discuss/84278/A-summary%3A-how-to-use-bit-manipulation-to-solve-problems-easily-and-efficiently)
- [DP patterns](https://leetcode.com/discuss/general-discussion/458695/Dynamic-Programming-Patterns)
- [Graph for beginners](https://leetcode.com/discuss/general-discussion/655708/Graph-For-Beginners-Problems-or-Pattern-or-Sample-Solutions)
- [Questions by pattern](https://leetcode.com/discuss/interview-question/448285/List-of-questions-sorted-by-common-patterns)
- [Trie template](https://leetcode.com/discuss/general-discussion/680706/article-on-trie-general-template-and-list-of-problems)
- [Binary search for beginners](https://leetcode.com/discuss/general-discussion/691825/binary-search-for-beginners-problems-patterns-sample-solutions)
- [Sliding window](https://leetcode.com/discuss/general-discussion/657507/Sliding-Window-for-Beginners-Problems-or-Template-or-Sample-Solutions)
- [Substring window template](https://leetcode.com/problems/minimum-window-substring/discuss/26808/Here-is-a-10-line-template-that-can-solve-most-'substring'-problems)
- [Monotonic queue](https://medium.com/algorithms-and-leetcode/monotonic-queue-explained-with-leetcode-problems-7db7c530c1d6)
