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

Common imports:

```java
import java.util.*;
```

### `ArrayList`

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

### `HashMap` — key/value lookup and frequency counts

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

### `HashSet` — membership and uniqueness

```java
Set<Integer> seen = new HashSet<>();
boolean addedForFirstTime = seen.add(x); // false if x was already present
boolean exists = seen.contains(x);
seen.remove(x);
int n = seen.size();
```

### `TreeSet` — sorted unique values and neighbors

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

### `PriorityQueue` — repeatedly take the smallest or largest

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

### `Queue` — FIFO processing

Use `ArrayDeque` as the usual queue implementation:

```java
Queue<Integer> queue = new ArrayDeque<>();
queue.offer(x);          // add to back
Integer front = queue.peek(); // inspect front; null if empty
Integer removed = queue.poll(); // remove front; null if empty
boolean empty = queue.isEmpty();
```

`offer`, `peek`, and `poll` are O(1) amortized. `poll` / `peek` return `null` on an empty queue; `remove` / `element` throw instead. The paired methods are `add`/`offer` (insert), `remove`/`poll` (remove front), and `element`/`peek` (inspect front); the first in each pair throws on failure, the second returns a special value (`false` or `null`).

### `Deque` / `ArrayDeque` — both ends, stack, or queue

**For stack (LIFO) operations, use `Deque` with `ArrayDeque`**—for example, `push`, `peek`, and `pop` below.

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
