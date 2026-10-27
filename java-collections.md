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
