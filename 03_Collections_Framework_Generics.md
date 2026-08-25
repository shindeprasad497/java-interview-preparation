# 03. Collections Framework, Generics & Sequenced Collections

> **Navigation**: [Master Index](README.md) | [Previous: Strings & Memory](02_Strings_Memory_Algorithms.md) | [Next: Modern Java Features](04_Modern_Java_Features_8_to_21.md)

---

## 📌 Chapter Overview
This module explores the complete Java Collections Framework architecture, internal data structures, `HashMap` and `ConcurrentHashMap` low-level mechanics, Generics with the PECS rule, and the **Java 21 Sequenced Collections** hierarchy.

---

## 1. Java Collections Framework Hierarchy

```
                               +-------------------+
                               |  <<Iterable<E>>>  |
                               +-------------------+
                                         |
                               +-------------------+
                               | <<Collection<E>>> |
                               +-------------------+
                                         |
         +-------------------------------+-------------------------------+
         |                               |                               |
 +---------------+               +---------------+               +---------------+
 |  <<List<E>>>  |               |  <<Set<E>>>   |               | <<Queue<E>>>  |
 +---------------+               +---------------+               +---------------+
   |           |                   |           |                   |           |
ArrayList   LinkedList          HashSet     TreeSet           PriorityQueue  ArrayDeque
                                   |                               |
                             LinkedHashSet                   <<Deque<E>>>
                                                                   |
                                                               LinkedList

 (Separate Map Hierarchy)
 +-------------------+
 |   <<Map<K, V>>>   |
 +-------------------+
   |        |        |
HashMap  TreeMap  LinkedHashMap
   |
ConcurrentHashMap (ConcurrentMap)
```

---

## 2. `HashMap` Internals (Java 8 to 21 Deep Dive)

### Q1. How does `HashMap` work internally? Explain Hashing, Bucketing, and Treeification.
**Answer:**

```
+-------------------------------------------------------------------------------+
|                            HASHMAP NODE ARRAY (TABLE)                         |
+-------------------------------------------------------------------------------+
|  Index 0:  null                                                               |
|  Index 1:  [ Node(hash, K1, V1) ] -> [ Node(hash, K2, V2) ] (LinkedList)       |
|  Index 2:  null                                                               |
|  Index 3:  [ TreeNode (Red-Black Tree Root) ] (Converted after 8 collisions)  |
|            /    \                                                             |
|         [TN]    [TN]                                                          |
|  Index 4:  [ Node(hash, K4, V4) ]                                             |
+-------------------------------------------------------------------------------+
```

#### Core Data Structures:
- **`Node<K, V>[] table`**: Underlying dynamic array of buckets (default initial capacity = 16, must always be a power of 2).
- **Collision Resolution**: Separate chaining via Singly Linked List $\rightarrow$ converted to **Red-Black Tree** (`TreeNode`) under high collisions.

#### Put Operation Step-by-Step:
1. **Hash Computation**: Secondary hash spreading to minimize collisions:
   ```java
   static final int hash(Object key) {
       int h;
       return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
   }
   ```
2. **Bucket Index Calculation**: Fast bitwise modulo operation (valid because table length $N$ is always a power of 2):
   $$\text{index} = (N - 1) \ \& \ \text{hash}$$
3. **Collision & Insertion**:
   - If bucket is empty $\rightarrow$ create new `Node`.
   - If collision exists:
     - Compare `node.hash == hash && (node.key == key || key.equals(node.key))`. If equal, replace value.
     - If bucket is a `TreeNode`, insert into the Red-Black Tree ($O(\log N)$).
     - Otherwise, traverse linked list. If not found, append at the tail.
4. **Treeification Thresholds**:
   - **TREEIFY_THRESHOLD (8)**: When a single bucket list length exceeds 8 elements.
   - **MIN_TREEIFY_CAPACITY (64)**: Treeification **only** occurs if total table capacity is $\ge 64$. If capacity $< 64$, the map **resizes** (`resize()`) instead of treeifying!
   - **UNTREEIFY_THRESHOLD (6)**: When elements in a tree shrink to $\le 6$ during resize, the tree converts back to a linked list.
5. **Rehashing & Resize**:
   - When `size > capacity * loadFactor` (default load factor = `0.75`), capacity doubles ($16 \rightarrow 32 \rightarrow 64$).
   - Elements are redistributed without recomputing full hashes: an element either stays at the same `index` or moves to `index + oldCapacity`.

---

## 3. `ConcurrentHashMap` Internals (Java 8+)

### Q2. How does `ConcurrentHashMap` achieve lock-free high concurrency?
**Answer:**

```
+-------------------------------------------------------------------------------+
|                       CONCURRENTHASHMAP LOCKING STRATEGY                      |
+-------------------------------------------------------------------------------+
|  Bucket 0 (Empty):     CAS (Compare-And-Swap) Insertion -> NO LOCK!           |
|                                                                               |
|  Bucket 1 (Populated): synchronized(FirstNode) {                              |
|                            // Locks ONLY Bucket 1; other buckets free!        |
|                        }                                                      |
|                                                                               |
|  Bucket 2 (Resizing):  ForwardingNode (hash = -1) -> Threads help resize!    |
|                                                                               |
|  Reads (get()):        Volatile reads -> 100% LOCK-FREE!                      |
+-------------------------------------------------------------------------------+
```

- **Java 7 vs Java 8+ Architecture**:
  - *Java 7*: Segment-based locking (`ReentrantLock` across 16 segments). Limited concurrency to 16 concurrent writers.
  - *Java 8+*: Removed segments. Uses **CAS (Compare-And-Swap)** for empty bucket insertion + synchronized lock on the **first Node of the bucket** for collisions.
- **Lock-Free Reads**: `get()` operations never lock! Bucket nodes use `volatile V val` and `volatile Node<K,V> next` references, ensuring instant visibility across CPU cores without locks.
- **Concurrent Resizing (Transfer)**: When expanding capacity, a special `ForwardingNode` (hash = `-1`) is placed in processed buckets. Reader threads bypass it, while writer threads assist in migrating remaining buckets.
- **CounterCell (High-Throughput Size Tracking)**: Instead of a single atomic integer for map size (which causes cache line contention), `ConcurrentHashMap` uses a striped `CounterCell[]` array (similar to `LongAdder`).

---

## 4. Fail-Fast vs. Fail-Safe (Weakly-Consistent) Iterators

### Q3. Explain Fail-Fast vs. Fail-Safe Iterators.
**Answer:**

| Feature | Fail-Fast | Fail-Safe / Weakly-Consistent |
| :--- | :--- | :--- |
| **Collections** | `ArrayList`, `HashSet`, `HashMap` | `CopyOnWriteArrayList`, `ConcurrentHashMap` |
| **Mechanism** | Checks `modCount != expectedModCount` on every `.next()` call | Iterates over a snapshot or live volatile node references |
| **Exception Thrown** | `ConcurrentModificationException` | None |
| **Allows Concurrent Mutation?**| No (unless using iterator's own `it.remove()`) | Yes |
| **Memory Overhead** | Zero extra memory | High for `CopyOnWriteArrayList` (copies entire array on write) |

---

## 5. Generics & The PECS Principle

### Q4. Explain Type Erasure and the PECS Principle.
**Answer:**

#### 1. Type Erasure:
Java Generics are a compile-time syntactic safety feature. During compilation, the JVM **erases** all generic type parameters and replaces them with their bounding class (`Object` for unbounded `<?>`, or the upper bound `T extends Number` $\rightarrow$ `Number`), inserting explicit casts where necessary.

#### 2. PECS Principle (Producer Extends, Consumer Super):
- **Producer (`? extends T`)**: If you only **READ** items from a collection, use `extends`. The collection *produces* data.
- **Consumer (`? super T`)**: If you only **WRITE** items into a collection, use `super`. The collection *consumes* data.

```java
public class CollectionsUtil {
    // Copies elements from source (Producer) to destination (Consumer)
    public static <T> void copy(List<? super T> dest, List<? extends T> src) {
        for (T item : src) {     // Reading from src (Producer Extends)
            dest.add(item);      // Writing into dest (Consumer Super)
        }
    }
}
```

---

## 6. Java 21 Sequenced Collections (JEP 431)

### Q5. What problem did Sequenced Collections solve in Java 21?
**Answer:**
Prior to Java 21, Java lacked a unified interface for collections with a defined encounter order:
- Getting the first element: `list.get(0)`, `deque.getFirst()`, `sortedSet.first()`.
- Getting the last element: `list.get(list.size() - 1)`, `deque.getLast()`, `sortedSet.last()`.
- Iterating in reverse: `Collections.reverse(list)` (mutates list!), or complex descending iterators.

#### Java 21 Solution:
Introduced 3 core interfaces: `SequencedCollection`, `SequencedSet`, and `SequencedMap`.

```
          <<Collection<E>>>
                 |
       <<SequencedCollection<E>>>
       - addFirst(E) / addLast(E)
       - getFirst()  / getLast()
       - removeFirst() / removeLast()
       - reversed() -> returns reverse-ordered view
         /               \
   <<List<E>>>      <<SequencedSet<E>>>
                    - LinkedHashSet
                    - TreeSet
```

```java
// Java 21 Sequenced Collections in Action
SequencedCollection<String> seq = new ArrayList<>();
seq.addLast("Alpha");
seq.addFirst("Omega");

System.out.println(seq.getFirst()); // "Omega"
System.out.println(seq.getLast());  // "Alpha"

// Reverse view without copying or mutating original collection!
SequencedCollection<String> reversedView = seq.reversed();
System.out.println(reversedView.getFirst()); // "Alpha"

// SequencedMap
SequencedMap<String, Integer> map = new LinkedHashMap<>();
map.putFirst("One", 1);
map.putLast("Ten", 10);
System.out.println(map.firstEntry()); // One=1
```

---

## 7. LRU Cache Implementation using `LinkedHashMap`

```java
import java.util.LinkedHashMap;
import java.util.Map;

public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int maxCapacity;

    public LRUCache(int capacity) {
        // accessOrder = true enables LRU ordering (most recently accessed placed at tail)
        super(capacity, 0.75f, true);
        this.maxCapacity = capacity;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        // Automatically evicts the least recently accessed item when size exceeds maxCapacity
        return size() > maxCapacity;
    }

    public static void main(String[] args) {
        LRUCache<Integer, String> cache = new LRUCache<>(3);
        cache.put(1, "A");
        cache.put(2, "B");
        cache.put(3, "C");
        cache.get(1);      // 1 accessed -> order becomes: 2, 3, 1
        cache.put(4, "D"); // Capacity exceeded -> 2 evicted!

        System.out.println(cache.keySet()); // Output: [3, 1, 4]
    }
}
```

---

## 8. Complete Collections Decision Matrix: When to Use What?

```
+----------------------------------------------------------------------------------------------------+
|                                MASTER COLLECTIONS SELECTION MATRIX                                 |
+----------------------------------------------------------------------------------------------------+
| Scenario / Requirement                    | Recommended Collection         | Key Rationale         |
| ----------------------------------------- | ------------------------------ | --------------------- |
| Fast random index access ($O(1)$)         | `ArrayList`                    | Contiguous memory array|
| Frequent insertions/deletions at HEAD/TAIL| `ArrayDeque` (or `LinkedList`) | Fast pointer updates  |
| Unique items with $O(1)$ lookup           | `HashSet`                      | Hash table bucketing  |
| Unique items preserving INSERTION ORDER   | `LinkedHashSet`                | Hash table + D-Linked |
| Unique items naturally SORTED ($O(\log N)$| `TreeSet`                      | Red-Black balanced tree|
| Key-Value pairs with $O(1)$ lookup        | `HashMap`                      | Fast array of buckets |
| Key-Value preserving INSERTION / LRU ORDER| `LinkedHashMap`                | Hash table + D-Linked |
| Key-Value naturally SORTED by Key         | `TreeMap`                      | Red-Black Tree        |
| Thread-Safe high-throughput Key-Value map | `ConcurrentHashMap`            | CAS + Striped locks   |
| High-performance Set of Enum constants    | `EnumSet`                      | Bit Vector / Bitmask  |
| High-performance Map of Enum keys         | `EnumMap`                      | Compact array index   |
| Standard LIFO Stack or FIFO Queue         | `ArrayDeque`                    | Circular array buffer |
| Priority-based retrieval (Min/Max Heap)   | `PriorityQueue`                | Binary Heap ($O(\log N)$|
| Thread-Safe Bounded Task Queue            | `ArrayBlockingQueue`           | Lock + Conditions     |
| Timed delayed task scheduling             | `DelayQueue`                   | Heap + Priority delay |
| Zero-capacity direct handoff queue        | `SynchronousQueue`             | Rendezvous channel    |
+----------------------------------------------------------------------------------------------------+
```

---

## 9. Comprehensive Element Breakdown & Specialized Collections

### Q6. Detail the specialized and concurrent collection implementations.
**Answer:**

1. **`EnumSet` & `EnumMap`**:
   - **`EnumSet`**: Implemented internally as a **Bit Vector** (single `long` bitmask for enums with $\le 64$ elements). Zero object allocation, extremely memory-efficient and faster than `HashSet`.
   - **`EnumMap`**: Backed by a flat array where the array index is determined by `enum.ordinal()`. Extremely fast $O(1)$ array lookups with zero hashing overhead.
2. **`WeakHashMap`**:
   - Keys are stored inside `WeakReference` wrappers.
   - When a key object has no remaining strong references, the Garbage Collector **automatically evicts the entry** from the map.
   - *Ideal Use Case*: In-memory metadata caches tied to object lifetimes.
3. **`IdentityHashMap`**:
   - Compares keys using **reference equality (`==`)** instead of `.equals()`.
   - *Ideal Use Case*: Serialization graph traversal, object cloning trees, compiler symbol tables.
4. **`CopyOnWriteArrayList` / `CopyOnWriteArraySet`**:
   - Every write operation (`add`, `set`, `remove`) creates a fresh **new copy** of the underlying array.
   - Reads are 100% lock-free and never throw `ConcurrentModificationException`.
   - *Ideal Use Case*: Read-heavy, write-rarely event listener lists.
5. **`ConcurrentSkipListMap` / `ConcurrentSkipListSet`**:
   - Concurrent, sorted collection backed by a **Skip List** data structure ($O(\log N)$).
   - Provides lock-free concurrent sorted access, range queries (`subMap`), and navigation.

---

## 10. Coding Patterns & Interview Problems Mapped per Data Structure

```
+----------------------------------------------------------------------------------------------------+
|                         DATA STRUCTURE TO CODING PATTERN INTERVIEW MAPPING                         |
+----------------------------------------------------------------------------------------------------+
```

### 1. `HashMap` & `HashSet` (Hash Table Patterns)
- **Core Pattern**: Fast $O(1)$ lookups, frequency counting, complement mapping, prefix sum caching.
- **Top Interview Coding Questions**:
  1. *Two Sum / 3Sum* (Map target complement).
  2. *Subarray Sum Equals K* (Prefix sum frequency map).
  3. *Group Anagrams* (Character frequency string as map key).
  4. *Longest Consecutive Sequence* (Set lookups in $O(N)$).

### 2. `TreeMap` & `TreeSet` (Red-Black Balanced BST Patterns)
- **Core Pattern**: Dynamic sorted order, range queries (`subSet`), floor/ceiling predecessor/successor lookups.
- **Top Interview Coding Questions**:
  1. *Meeting Rooms II* (Chronological event timestamp count using `TreeMap`).
  2. *My Calendar I / II / III* (Interval overlap booking via `floorKey` / `ceilingKey`).
  3. *Contains Duplicate III* (`TreeSet.ceiling()` within sliding window).

### 3. `PriorityQueue` (Min-Heap / Max-Heap Patterns)
- **Core Pattern**: Finding Extremes, Top-K elements, streaming medians, merging sorted streams.
- **Top Interview Coding Questions**:
  1. *Top K Frequent Elements* (Min-Heap of size $K$).
  2. *Find Median from Data Stream* (Two Heaps: Max-Heap for lower half + Min-Heap for upper half).
  3. *Merge K Sorted Lists* (Min-Heap storing heads of all $K$ lists).
  4. *Task Scheduler / Reorganize String* (Max-Heap with cooldown queues).

### 4. `ArrayDeque` (Stack, Queue & Monotonic Queue Patterns)
- **Core Pattern**: LIFO Stack, FIFO Queue, BFS graph traversals, Monotonic decreasing/increasing windows.
- **Top Interview Coding Questions**:
  1. *Sliding Window Maximum* (Monotonic decreasing `ArrayDeque` maintaining potential maximums in $O(N)$).
  2. *Daily Temperatures / Next Greater Element* (Monotonic Stack).
  3. *Largest Rectangle in Histogram* (Monotonic Stack tracking boundary indices).
  4. *Binary Tree Level Order Traversal* (BFS queue).

### 5. `LinkedHashMap` (Order-Preserving & Eviction Patterns)
- **Core Pattern**: Access-order LRU cache, insertion-order deduplication.
- **Top Interview Coding Questions**:
  1. *LRU Cache (Least Recently Used)* (`LinkedHashMap.removeEldestEntry`).
  2. *LFU Cache (Least Frequently Used)* (Frequency map + LinkedHashSet per frequency).
  3. *First Unique Character in a Stream*.

---

> **Next Chapter**: [04 Modern Java Features (Java 8 to 21 LTS)](04_Modern_Java_Features_8_to_21.md)

