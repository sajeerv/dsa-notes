# Java collections

Imported and cleaned from `Java Collections Cheat Sheets.pages`. Complexities are typical / amortized average case unless noted. `O(h / n)` for “next” is iterator cost when the table is sparse (`h` = capacity, `n` = size).

## When to pick what

| Need | Prefer |
| --- | --- |
| Index access, append | `ArrayList` |
| Frequent insert/delete at ends, queue/deque | `ArrayDeque` (or `LinkedList` if you already have one) |
| Unique keys, no order | `HashSet` / `HashMap` |
| Unique keys, insertion order | `LinkedHashSet` / `LinkedHashMap` |
| Sorted keys | `TreeSet` / `TreeMap` |
| Enum keys | `EnumSet` / `EnumMap` |
| Priority / k-th | `PriorityQueue` |
| LRU-style iteration + O(1) lookup | `LinkedHashMap` (access-order) |

`LinkedHashMap` vs `TreeMap` vs `HashMap`:

| | `HashMap` | `LinkedHashMap` | `TreeMap` |
| --- | --- | --- | --- |
| Iteration | No order | Insertion (or access) order | Sorted by key |
| Null keys | Yes | Yes | No (keys are compared) |

## LeetCode operations quick reference

Organized by the **data structure you need**; each heading names the Java collection or representation to use.

Common imports:

```java
import java.util.*;
```

### Resizable array / indexed sequence — `ArrayList`

```java
List<Integer> list = new ArrayList<>();
list.add(10);                 // append
list.add(0, 5);               // insert at index; shifts later elements
int value = list.get(0);
list.set(0, 7);               // replace; returns old value
list.remove(list.size() - 1); // remove by index
list.remove(Integer.valueOf(10)); // remove by value
boolean has = list.contains(7);
int n = list.size();
boolean empty = list.isEmpty();
Collections.sort(list);       // ascending
Collections.sort(list, Comparator.reverseOrder());
```

`get` / `set` are O(1); inserting or removing away from the end is O(n). For `List<Integer>`, `remove(1)` means remove index 1; use `remove(Integer.valueOf(1))` to remove the value 1.

### Hash table for key/value lookup — `HashMap`

```java
Map<Integer, Integer> map = new HashMap<>();
map.put(key, value);
int valueOrDefault = map.getOrDefault(key, 0);
map.put(key, map.getOrDefault(key, 0) + 1); // frequency count
map.merge(key, 1, Integer::sum);            // same frequency update
boolean hasKey = map.containsKey(key);
int value = map.get(key);                   // null if absent (for Integer values)
map.remove(key);

for (Map.Entry<Integer, Integer> entry : map.entrySet()) {
    int k = entry.getKey();
    int v = entry.getValue();
}
```

Use `containsKey` or `getOrDefault` when a key might be absent; `get` can return `null`.

### Hash table for membership / uniqueness — `HashSet`

```java
Set<Integer> seen = new HashSet<>();
boolean addedForFirstTime = seen.add(x); // false if x was already present
boolean exists = seen.contains(x);
seen.remove(x);
int n = seen.size();
```

### Ordered set / sorted unique values — `TreeSet`

```java
NavigableSet<Integer> sorted = new TreeSet<>();
sorted.add(x);
sorted.remove(x);
boolean exists = sorted.contains(x);

Integer lower = sorted.lower(x);       // greatest value < x
Integer floor = sorted.floor(x);       // greatest value <= x
Integer ceiling = sorted.ceiling(x);   // smallest value >= x
Integer higher = sorted.higher(x);     // smallest value > x
Integer first = sorted.first();        // smallest; throws if empty
Integer last = sorted.last();          // largest; throws if empty
Integer smallestRemoved = sorted.pollFirst();
Integer largestRemoved = sorted.pollLast();
```

Neighbor methods return `null` if no such value exists. Add, remove, contains, and neighbor lookups are O(log n).

### Heap / priority queue — `PriorityQueue`

```java
PriorityQueue<Integer> minHeap = new PriorityQueue<>();
minHeap.offer(x);             // add
Integer smallest = minHeap.peek(); // inspect; null if empty
Integer removed = minHeap.poll();  // remove smallest; null if empty

PriorityQueue<Integer> maxHeap =
    new PriorityQueue<>(Comparator.reverseOrder());
maxHeap.offer(x);
Integer largest = maxHeap.poll();
```

`offer` / `poll` are O(log n), `peek` is O(1). Iterating a `PriorityQueue` does **not** produce sorted order; repeatedly `poll` to consume values in priority order.

For custom items, provide a comparator. Use `Integer.compare` / `Comparator.comparingInt` rather than subtracting values, which can overflow:

```java
PriorityQueue<int[]> bySecond =
    new PriorityQueue<>((a, b) -> Integer.compare(a[1], b[1]));
```

### Queue / FIFO processing — `Queue` with `ArrayDeque`

Use `ArrayDeque` as the usual queue implementation:

```java
Queue<Integer> queue = new ArrayDeque<>();
queue.offer(x);          // add to back
Integer front = queue.peek(); // inspect front; null if empty
Integer removed = queue.poll(); // remove front; null if empty
boolean empty = queue.isEmpty();
```

`offer`, `peek`, and `poll` are O(1) amortized. `poll` / `peek` return `null` on an empty queue; `remove` / `element` throw instead. The paired methods are `add`/`offer` (insert), `remove`/`poll` (remove front), and `element`/`peek` (inspect front); the first in each pair throws on failure, the second returns a special value (`false` or `null`).

### Deque / both ends, stack, or queue — `Deque` with `ArrayDeque`

**Use `Deque` with `ArrayDeque` for stack (LIFO) operations; don’t use the legacy `Stack` class.** For queue operations, use `offer`, `poll`, and `peek` as in the Queue section above.

```java
Deque<Integer> deque = new ArrayDeque<>();
deque.addFirst(x);       // or offerFirst
deque.addLast(y);        // or offerLast
Integer front = deque.peekFirst();
Integer back = deque.peekLast();
Integer firstRemoved = deque.pollFirst();
Integer lastRemoved = deque.pollLast();

deque.push(x);           // stack push: same as addFirst
Integer top = deque.peek(); // stack top: same as peekFirst
Integer popped = deque.pop(); // same as removeFirst; throws if empty
```

`ArrayDeque` is a common choice for BFS queues, monotonic deques, and stacks. It does not allow `null` elements.

#### Monotonic stack or deque — `Deque` with `ArrayDeque`

Use when you need the next greater/smaller element or a sliding-window minimum/maximum. Keep candidate indices in increasing or decreasing value order by removing dominated indices from the end.

```java
Deque<Integer> indices = new ArrayDeque<>();
for (int i = 0; i < nums.length; i++) {
    while (!indices.isEmpty() && nums[indices.peekLast()] <= nums[i]) {
        indices.pollLast();
    }
    // indices.peekLast() is the previous greater element's index, if any
    indices.addLast(i);
}
```

### Ordered map / sorted keys and ranges — `TreeMap`

Use when keys must stay sorted or you need the closest key or a key range.

```java
NavigableMap<Integer, String> map = new TreeMap<>();
map.put(key, value);
Integer lowerKey = map.lowerKey(key);     // greatest key < key
Integer floorKey = map.floorKey(key);     // greatest key <= key
Integer ceilingKey = map.ceilingKey(key); // smallest key >= key
Integer higherKey = map.higherKey(key);   // smallest key > key
Map<Integer, String> range = map.subMap(from, true, to, false);
```

### Graph / adjacency list — `ArrayList` of neighbor lists

For sparse graphs, store a neighbor list for each vertex. For weighted graphs, store edge objects or `int[]{neighbor, weight}`.

```java
int n = numberOfVertices;
List<List<Integer>> graph = new ArrayList<>();
for (int i = 0; i < n; i++) graph.add(new ArrayList<>());

graph.get(u).add(v); // directed edge u -> v
graph.get(v).add(u); // also add this edge for an undirected graph
```

For traversal, use `boolean[] visited`; use `Queue<Integer> queue = new ArrayDeque<>()` for BFS.

### Disjoint sets / connectivity — Union-Find arrays

Use to track connected components as edges are added. Store `parent` and `size` (or `rank`) in primitive arrays; combine path compression with union by size/rank.

```java
int[] parent = new int[n];
int[] size = new int[n];
for (int i = 0; i < n; i++) {
    parent[i] = i;
    size[i] = 1;
}

int find(int x) {
    if (parent[x] != x) parent[x] = find(parent[x]);
    return parent[x];
}

boolean union(int a, int b) {
    int ra = find(a), rb = find(b);
    if (ra == rb) return false;
    if (size[ra] < size[rb]) { int t = ra; ra = rb; rb = t; }
    parent[rb] = ra;
    size[ra] += size[rb];
    return true;
}
```

Put `find` and `union` inside a class or a helper class; Java does not allow methods nested inside another method.

### Trie / prefix tree — custom nodes

Use for prefix search and word dictionaries. For lowercase English letters, a 26-child array is simple and compact:

```java
class TrieNode {
    TrieNode[] children = new TrieNode[26];
    boolean isWord;
}
```

For larger or sparse alphabets, store children in a `Map<Character, TrieNode>` instead.

### Binary tree / BST — custom nodes

LeetCode usually provides a `TreeNode` class. Otherwise, define a node with a value and left/right references; use recursion or an explicit `Deque<TreeNode>` for traversals.

```java
class TreeNode {
    int val;
    TreeNode left, right;
    TreeNode(int val) { this.val = val; }
}
```

For a BST, the same node structure applies with the ordering rule (values in the left subtree are smaller; values in the right subtree are larger). For sorted key lookup with built-in operations, `TreeMap` / `TreeSet` are often more convenient.

### Prefix sums, grids, and dense indexed data — primitive arrays

Use primitive arrays when indexes are dense and known. Prefix sums make range-sum queries O(1) after O(n) preprocessing:

```java
long[] prefix = new long[n + 1];
for (int i = 0; i < n; i++) prefix[i + 1] = prefix[i] + nums[i];
long rangeSum = prefix[r + 1] - prefix[l]; // inclusive range l..r
```

For a grid, use `int[][]` / `char[][]`; use `boolean[][] visited` for traversal state, or mark cells in place when allowed.

### Range-query tree — Fenwick tree or segment tree arrays

When values change and you need many range-sum queries, a Fenwick tree is compact and supports point updates and prefix sums in O(log n):

```java
int[] bit = new int[n + 1]; // 1-indexed
void add(int index, int delta) {
    for (int i = index + 1; i < bit.length; i += i & -i) bit[i] += delta;
}
int prefixSum(int endExclusive) { // sum of original indices [0, endExclusive)
    int sum = 0;
    for (int i = endExclusive; i > 0; i -= i & -i) sum += bit[i];
    return sum;
}
```

Use a segment tree (usually a custom array-backed tree) for more general range queries and updates, such as range minimum/maximum or custom merge operations. For a static array with no updates, a prefix-sum array is simpler.

### Bit set / flags — integer mask or `BitSet`

Use an integer mask for a small fixed number of flags; use `BitSet` for a larger dynamic collection of bits.

```java
int mask = 0;
mask |= 1 << bit;                        // set bit
boolean isSet = (mask & (1 << bit)) != 0; // test bit
mask &= ~(1 << bit);                     // clear bit

BitSet bits = new BitSet();
bits.set(index);
boolean present = bits.get(index);
bits.clear(index);
```

### Linked list / node-by-node sequence — custom nodes or `LinkedList`

For problems that give you `ListNode` (reversing a list, merging lists, detecting a cycle), work with the provided nodes and pointers. Java's `LinkedList` is rarely the best choice for LeetCode: indexed access is O(n). For a sequence where you only need to add/remove at the ends, prefer `ArrayDeque`.

## List complexity

| Type | Add | Remove | Get | Contains | Next | Structure |
| --- | --- | --- | --- | --- | --- | --- |
| `ArrayList` | O(1) amortized | O(n) | O(1) | O(n) | O(1) | Array |
| `LinkedList` | O(1) | O(1)* | O(n) | O(n) | O(1) | Linked list |
| `CopyOnWriteArrayList` | O(n) | O(n) | O(1) | O(n) | O(1) | Array |

\*O(1) if you already have the node / use iterator remove; index-based remove is O(n).

`ArrayList`: `size`, `isEmpty`, `iterator`, and `listIterator` are O(1). `add` at the end is amortized O(1). Not synchronized.

## Set complexity

| Type | Add | Remove | Contains | Next | Size | Structure |
| --- | --- | --- | --- | --- | --- | --- |
| `HashSet` | O(1) | O(1) | O(1) | O(h / n) | O(1) | Hash table |
| `LinkedHashSet` | O(1) | O(1) | O(1) | O(1) | O(1) | Hash table + linked list |
| `EnumSet` | O(1) | O(1) | O(1) | O(1) | O(1) | Bit vector |
| `TreeSet` | O(log n) | O(log n) | O(log n) | O(log n) | O(1) | Red-black tree |
| `CopyOnWriteArraySet` | O(n) | O(n) | O(n) | O(1) | O(1) | Array |
| `ConcurrentSkipListSet` | O(log n) | O(log n) | O(log n) | O(1) | O(n) | Skip list |

`LinkedHashSet` keeps insertion order and still searches in expected O(1).

## Queue / deque complexity

| Type | Offer | Peek | Poll | Remove | Size | Structure |
| --- | --- | --- | --- | --- | --- | --- |
| `PriorityQueue` | O(log n) | O(1) | O(log n) | O(n) | O(1) | Binary heap |
| `LinkedList` | O(1) | O(1) | O(1) | O(n) | O(1) | Linked list |
| `ArrayDeque` | O(1) | O(1) | O(1) | O(n) | O(1) | Circular array |
| `ConcurrentLinkedQueue` | O(1) | O(1) | O(1) | O(n) | O(n) | Linked list |
| `ArrayBlockingQueue` | O(1) | O(1) | O(1) | O(n) | O(1) | Array |
| `PriorityBlockingQueue` | O(log n) | O(1) | O(log n) | O(n) | O(1) | Binary heap |
| `SynchronousQueue` | O(1) | O(1) | O(1) | O(n) | O(1) | No storage |
| `DelayQueue` | O(log n) | O(1) | O(log n) | O(n) | O(1) | Binary heap |
| `LinkedBlockingQueue` | O(1) | O(1) | O(1) | O(n) | O(1) | Linked list |

Prefer `ArrayDeque` as a stack or queue:

```java
Deque<Integer> stack = new ArrayDeque<>();
stack.push(x);
stack.pop();
stack.peek();
```

## Map complexity

| Type | Get | ContainsKey | Next | Structure |
| --- | --- | --- | --- | --- |
| `HashMap` | O(1) | O(1) | O(h / n) | Hash table |
| `LinkedHashMap` | O(1) | O(1) | O(1) | Hash table + linked list |
| `IdentityHashMap` | O(1) | O(1) | O(h / n) | Array (`==` keys) |
| `WeakHashMap` | O(1) | O(1) | O(h / n) | Hash table (weak keys) |
| `EnumMap` | O(1) | O(1) | O(1) | Array |
| `TreeMap` | O(log n) | O(log n) | O(log n) | Red-black tree |
| `ConcurrentHashMap` | O(1) | O(1) | O(h / n) | Hash tables |
| `ConcurrentSkipListMap` | O(log n) | O(log n) | O(1) | Skip list |

Reverse-order view of a `TreeMap`:

```java
NavigableSet<K> keys = treeMap.descendingKeySet();
```

Also learn `lowerKey`, `floorKey`, `ceilingKey`, `higherKey`, and `subMap`.

## HashMap internals

**Direct addressing:** when the key universe is small and keys are unique, a plain array (direct-address table) is enough.

**Hashing:** a hash function maps a key to a bucket.

- **Initial capacity** — number of buckets at creation.
- **Load factor** — how full the table may get before resize. Default `0.75` is a time/space tradeoff.
- Resize (rehash) happens when `size > capacity * loadFactor`. Set initial capacity from expected size if you know it.

**Collision resolution (chaining):** colliding keys share a list (Java 8+ can treeify long buckets).

| Op | Cost |
| --- | --- |
| `put` / `get` | O(k) in a chain of length k; expected O(1) |
| `remove` | O(k); O(1) if the chain is a doubly linked structure and you have the node |

Custom keys in a `HashSet` / `HashMap`: override **both** `equals` and `hashCode`. With several fields, combine their hash codes (XOR is a simple option).

Do not compare `Integer` objects with `==`; use `equals`.

## List and array conversions

`Arrays.asList` / `Collections.addAll` do **not** work on primitive arrays (`int[]`).

```java
// array -> list (fixed-size view of the array)
String[] a = { "A", "B", "C", "D" };
List<String> list = Arrays.asList(a);

List<String> copy = new ArrayList<>();
Collections.addAll(copy, a);

List<Integer> fromValues = Arrays.asList(4, 5, 6);

// list -> array (works for any Collection)
Integer[] arr = al.toArray(new Integer[0]);
String[] sarr = list.toArray(new String[0]);
```

Replace, print, sublist:

```java
al.set(1, 40);
System.out.println(Arrays.toString(list.toArray()));

// fromIndex inclusive, toIndex exclusive; backed by the original list
List<E> view = list.subList(2, 5);
List<E> independent = new ArrayList<>(list.subList(2, 5));
```

## Sorting and searching

```java
Collections.sort(personList, new Comparator<Person>() {
    public int compare(Person p1, Person p2) {
        return p1.getName().compareTo(p2.getName());
    }
});

Collections.sort(numbers, (a, b) -> (b + a).compareTo(a + b));
Arrays.sort(intervals, (i1, i2) -> Integer.compare(i1[0], i2[0]));
Collections.sort(list, Comparator.comparingLong(Message::getTime));

Arrays.sort(arr, Collections.reverseOrder());
Arrays.sort(arr, Comparator.reverseOrder());
```

Built-in sorts:

- No comparator: dual-pivot quicksort on primitives; **not stable**.
- With a `Comparator` on objects: TimSort / merge-style; **stable**.

```java
int i = Collections.binarySearch(list, key);              // list must be ascending
int j = Collections.binarySearch(list, key, comparator);
// not found: returns -(insertionPoint) - 1
// example: -6 means insertion point 5
```

Also: `Collections.min` / `max` (optional comparator), `Collections.disjoint`.

## Iterators

`ListIterator` can walk a list **either direction**, modify during iteration, and report the current index. `Iterator` is forward-only.

## Heap index math (`PriorityQueue` is a binary heap)

For 0-based index `i`:

- left = `2i + 1`
- right = `2i + 2`
- parent = `(i - 1) / 2`
- leaves start around `n / 2`
