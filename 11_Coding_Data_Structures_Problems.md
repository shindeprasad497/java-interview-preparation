# 11. Coding, Data Structures & Concurrency Challenges (15 Master Problems)

> **Navigation**: [Master Index](README.md) | [Previous: JVM Troubleshooting](10_JVM_Troubleshooting_Dumps_Profiling.md) | [Next: SOLID Principles & Design Patterns](12_SOLID_Principles_Design_Patterns.md)

---

## 📌 Chapter Overview
A curated collection of **15 high-frequency coding problems from Basic to Advanced** tested in **Senior & Lead Java Technical Interviews**. Covers Arrays, Strings, Stacks, Queues, Heaps, Trees, LRU/LFU Caches, Concurrency, and Stream API transformations.

---

## 1. Hash Tables & Prefix Sum

### Problem 1: Subarray Sum Equals K
*Given an array of integers `nums` and an integer `k`, return the total number of subarrays whose sum equals to `k`.*

```java
public class SubarraySumEqualsK {
    public static int subarraySum(int[] nums, int k) {
        int count = 0, currentPrefixSum = 0;
        // Map stores: prefixSum -> frequency of occurrence
        Map<Integer, Integer> prefixSumFreq = new HashMap<>();
        prefixSumFreq.put(0, 1); // Base case for subarrays starting at index 0

        for (int num : nums) {
            currentPrefixSum += num;
            // If (currentPrefixSum - k) exists in map, we found valid subarrays!
            if (prefixSumFreq.containsKey(currentPrefixSum - k)) {
                count += prefixSumFreq.get(currentPrefixSum - k);
            }
            prefixSumFreq.put(currentPrefixSum, prefixSumFreq.getOrDefault(currentPrefixSum, 0) + 1);
        }
        return count;
    }
}
```
- **Time Complexity**: $O(N)$ single pass.
- **Space Complexity**: $O(N)$ hash map storage.

---

## 2. Two Pointers & Sliding Window

### Problem 2: Longest Substring Without Repeating Characters
*Find the length of the longest substring without duplicate characters.*

```java
public class LongestUniqueSubstring {
    public static int lengthOfLongestSubstring(String s) {
        if (s == null || s.isEmpty()) return 0;
        int maxLength = 0, left = 0;
        Map<Character, Integer> lastSeen = new HashMap<>();

        for (int right = 0; right < s.length(); right++) {
            char c = s.charAt(right);
            if (lastSeen.containsKey(c)) {
                left = Math.max(left, lastSeen.get(c)); // Jump left pointer past duplicate
            }
            lastSeen.put(c, right + 1);
            maxLength = Math.max(maxLength, right - left + 1);
        }
        return maxLength;
    }
}
```
- **Time Complexity**: $O(N)$.
- **Space Complexity**: $O(\min(N, M))$ where $M$ is character set size.

---

## 3. Stacks & Monotonic Data Structures

### Problem 3: Min Stack ($O(1)$ Time)
*Design a stack that supports `push`, `pop`, `top`, and retrieving the minimum element in $O(1)$ time.*

```java
public class MinStack {
    private final Deque<Integer> mainStack = new ArrayDeque<>();
    private final Deque<Integer> minStack = new ArrayDeque<>();

    public void push(int val) {
        mainStack.push(val);
        if (minStack.isEmpty() || val <= minStack.peek()) {
            minStack.push(val);
        }
    }

    public void pop() {
        if (mainStack.isEmpty()) return;
        int removed = mainStack.pop();
        if (removed == minStack.peek()) {
            minStack.pop();
        }
    }

    public int top() { return mainStack.peek(); }
    public int getMin() { return minStack.peek(); }
}
```

---

## 4. Concurrency & Synchronization

### Problem 4: Thread-Safe Singleton
```java
// Bill Pugh Initialization-on-demand Holder Idiom (Recommended)
public class BillPughSingleton {
    private BillPughSingleton() {}

    private static class InstanceHolder {
        private static final BillPughSingleton INSTANCE = new BillPughSingleton();
    }

    public static BillPughSingleton getInstance() {
        return InstanceHolder.INSTANCE; // Thread-safe with zero lock synchronization overhead!
    }
}
```

---

### Problem 5: Custom Bounded `BlockingQueue`
```java
public class CustomBoundedBlockingQueue<T> {
    private final Object[] items;
    private int count = 0, putIndex = 0, takeIndex = 0;
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();

    public CustomBoundedBlockingQueue(int capacity) {
        this.items = new Object[capacity];
    }

    public void put(T item) throws InterruptedException {
        lock.lock();
        try {
            while (count == items.length) notFull.await();
            items[putIndex] = item;
            if (++putIndex == items.length) putIndex = 0;
            count++;
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }

    @SuppressWarnings("unchecked")
    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0) notEmpty.await();
            T item = (T) items[takeIndex];
            items[takeIndex] = null;
            if (++takeIndex == items.length) takeIndex = 0;
            count--;
            notFull.signal();
            return item;
        } finally {
            lock.unlock();
        }
    }
}
```

---

## 5. Cache Eviction Data Structures

### Problem 6: LRU Cache (Least Recently Used)
```java
public class LRUCache<K, V> {
    private static class Node<K, V> {
        K key; V value;
        Node<K, V> prev, next;
        Node(K k, V v) { this.key = k; this.value = v; }
    }

    private final int capacity;
    private final Map<K, Node<K, V>> map = new HashMap<>();
    private final Node<K, V> head = new Node<>(null, null);
    private final Node<K, V> tail = new Node<>(null, null);

    public LRUCache(int capacity) {
        this.capacity = capacity;
        head.next = tail; tail.prev = head;
    }

    public synchronized V get(K key) {
        Node<K, V> node = map.get(key);
        if (node == null) return null;
        moveToHead(node);
        return node.value;
    }

    public synchronized void put(K key, V value) {
        Node<K, V> node = map.get(key);
        if (node != null) {
            node.value = value;
            moveToHead(node);
        } else {
            if (map.size() >= capacity) {
                Node<K, V> lru = removeTail();
                map.remove(lru.key);
            }
            Node<K, V> newNode = new Node<>(key, value);
            map.put(key, newNode);
            addToHead(newNode);
        }
    }

    private void addToHead(Node<K, V> node) {
        node.next = head.next; node.prev = head;
        head.next.prev = node; head.next = node;
    }

    private void removeNode(Node<K, V> node) {
        node.prev.next = node.next; node.next.prev = node.prev;
    }

    private void moveToHead(Node<K, V> node) { removeNode(node); addToHead(node); }
    private Node<K, V> removeTail() { Node<K, V> res = tail.prev; removeNode(res); return res; }
}
```

---

### Problem 7: LFU Cache (Least Frequently Used - $O(1)$ Time)
```java
public class LFUCache {
    private final int capacity;
    private int minFreq = 0;
    private final Map<Integer, Integer> keyToVal = new HashMap<>();
    private final Map<Integer, Integer> keyToFreq = new HashMap<>();
    private final Map<Integer, LinkedHashSet<Integer>> freqToKeys = new HashMap<>();

    public LFUCache(int capacity) { this.capacity = capacity; }

    public int get(int key) {
        if (!keyToVal.containsKey(key)) return -1;
        updateFrequency(key);
        return keyToVal.get(key);
    }

    public void put(int key, int value) {
        if (capacity <= 0) return;
        if (keyToVal.containsKey(key)) {
            keyToVal.put(key, value);
            updateFrequency(key);
            return;
        }
        if (keyToVal.size() >= capacity) {
            int evictKey = freqToKeys.get(minFreq).iterator().next(); // Evict LRU within min frequency
            freqToKeys.get(minFreq).remove(evictKey);
            keyToVal.remove(evictKey);
            keyToFreq.remove(evictKey);
        }
        keyToVal.put(key, value);
        keyToFreq.put(key, 1);
        freqToKeys.computeIfAbsent(1, k -> new LinkedHashSet<>()).add(key);
        minFreq = 1;
    }

    private void updateFrequency(int key) {
        int freq = keyToFreq.get(key);
        keyToFreq.put(key, freq + 1);
        freqToKeys.get(freq).remove(key);
        if (freqToKeys.get(freq).isEmpty() && freq == minFreq) minFreq++;
        freqToKeys.computeIfAbsent(freq + 1, k -> new LinkedHashSet<>()).add(key);
    }
}
```

---

## 6. Heaps & Priority Queues

### Problem 8: Merge K Sorted Lists
```java
public class MergeKSortedLists {
    public static class ListNode {
        int val; ListNode next;
        ListNode(int v) { this.val = v; }
    }

    public ListNode mergeKLists(ListNode[] lists) {
        if (lists == null || lists.length == 0) return null;

        // Min-Heap ordered by node value
        PriorityQueue<ListNode> minHeap = new PriorityQueue<>(Comparator.comparingInt(a -> a.val));
        for (ListNode root : lists) {
            if (root != null) minHeap.offer(root);
        }

        ListNode dummy = new ListNode(0);
        ListNode current = dummy;

        while (!minHeap.isEmpty()) {
            ListNode smallest = minHeap.poll();
            current.next = smallest;
            current = current.next;
            if (smallest.next != null) {
                minHeap.offer(smallest.next);
            }
        }
        return dummy.next;
    }
}
```

---

### Problem 9: Top K Frequent Elements
```java
public class TopKFrequentElements {
    public int[] topKFrequent(int[] nums, int k) {
        Map<Integer, Integer> countMap = new HashMap<>();
        for (int n : nums) countMap.put(n, countMap.getOrDefault(n, 0) + 1);

        // Min-Heap storing map keys based on frequency
        PriorityQueue<Integer> minHeap = new PriorityQueue<>(Comparator.comparingInt(countMap::get));

        for (int key : countMap.keySet()) {
            minHeap.offer(key);
            if (minHeap.size() > k) minHeap.poll(); // Keep heap size bounded to K
        }

        int[] result = new int[k];
        for (int i = k - 1; i >= 0; i--) result[i] = minHeap.poll();
        return result;
    }
}
```

---

## 7. Advanced Monotonic Deques & Sweepline Intervals

### Problem 10: Sliding Window Maximum ($O(N)$ Time)
```java
public class SlidingWindowMax {
    public int[] maxSlidingWindow(int[] nums, int k) {
        if (nums == null || nums.length == 0) return new int[0];
        int n = nums.length;
        int[] result = new int[n - k + 1];
        int ri = 0;
        // Monotonic decreasing double-ended queue storing array INDICES
        Deque<Integer> deque = new ArrayDeque<>();

        for (int i = 0; i < n; i++) {
            // 1. Remove indices out of current sliding window
            while (!deque.isEmpty() && deque.peekFirst() < i - k + 1) {
                deque.pollFirst();
            }
            // 2. Remove smaller elements as they cannot be the maximum
            while (!deque.isEmpty() && nums[deque.peekLast()] < nums[i]) {
                deque.pollLast();
            }
            deque.offerLast(i);

            // 3. Append current maximum to result
            if (i >= k - 1) {
                result[ri++] = nums[deque.peekFirst()];
            }
        }
        return result;
    }
}
```

---

### Problem 11: Meeting Rooms II (Minimum Conference Rooms)
```java
public class MeetingRoomsII {
    public int minMeetingRooms(int[][] intervals) {
        if (intervals == null || intervals.length == 0) return 0;
        // Sort intervals by start time
        Arrays.sort(intervals, Comparator.comparingInt(a -> a[0]));

        // Min-Heap stores meeting END times
        PriorityQueue<Integer> minHeap = new PriorityQueue<>();
        minHeap.offer(intervals[0][1]);

        for (int i = 1; i < intervals.length; i++) {
            // If current meeting starts after or when earliest meeting ends -> reuse room
            if (intervals[i][0] >= minHeap.peek()) {
                minHeap.poll();
            }
            minHeap.offer(intervals[i][1]);
        }
        return minHeap.size(); // Total active rooms required
    }
}
```

---

## 8. Trees & Trie (Prefix Tree)

### Problem 12: Implement Trie (Prefix Tree) with Autocomplete
```java
public class Trie {
    private static class TrieNode {
        Map<Character, TrieNode> children = new HashMap<>();
        boolean isEndOfWord = false;
    }

    private final TrieNode root = new TrieNode();

    public void insert(String word) {
        TrieNode curr = root;
        for (char c : word.toCharArray()) {
            curr = curr.children.computeIfAbsent(c, k -> new TrieNode());
        }
        curr.isEndOfWord = true;
    }

    public boolean search(String word) {
        TrieNode node = findNode(word);
        return node != null && node.isEndOfWord;
    }

    public boolean startsWith(String prefix) {
        return findNode(prefix) != null;
    }

    private TrieNode findNode(String str) {
        TrieNode curr = root;
        for (char c : str.toCharArray()) {
            curr = curr.children.get(c);
            if (curr == null) return null;
        }
        return curr;
    }
}
```

---

## 9. Dynamic Streaming Data

### Problem 13: Find Median from Data Stream (Two Heaps)
```java
public class MedianFinder {
    private final PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Collections.reverseOrder()); // Lower half
    private final PriorityQueue<Integer> minHeap = new PriorityQueue<>();                            // Upper half

    public void addNum(int num) {
        if (maxHeap.isEmpty() || num <= maxHeap.peek()) {
            maxHeap.offer(num);
        } else {
            minHeap.offer(num);
        }

        // Balance heaps so size difference is at most 1
        if (maxHeap.size() > minHeap.size() + 1) {
            minHeap.offer(maxHeap.poll());
        } else if (minHeap.size() > maxHeap.size()) {
            maxHeap.offer(minHeap.poll());
        }
    }

    public double findMedian() {
        if (maxHeap.size() == minHeap.size()) {
            return (maxHeap.peek() + minHeap.peek()) / 2.0;
        }
        return maxHeap.peek();
    }
}
```

---

## 10. Concurrency Deadlocks & Stream Aggregations

### Problem 14: Deadlock Simulation & Consistent Lock Ordering Fix
```java
public class DeadlockResolution {
    private final Object lock1 = new Object();
    private final Object lock2 = new Object();

    // ❌ VULNERABLE: Inconsistent lock acquisition order causes Deadlock!
    public void executeVulnerableA() { synchronized (lock1) { synchronized (lock2) {} } }
    public void executeVulnerableB() { synchronized (lock2) { synchronized (lock1) {} } }

    // ✅ RESOLUTION: Enforce Global Consistent Lock Ordering via System Identity Hash
    public void executeSafe(Object a, Object b) {
        Object first = System.identityHashCode(a) < System.identityHashCode(b) ? a : b;
        Object second = (first == a) ? b : a;

        synchronized (first) {
            synchronized (second) {
                // Guaranteed deadlock-free!
            }
        }
    }
}
```

---

### Problem 15: Complex Stream Grouping, Partitioning & Top-N
```java
public record Transaction(String txId, String department, double amount, String currency) {}

public class StreamAnalytics {

    // 1. Group transactions by department and calculate total revenue in USD
    public Map<String, Double> totalRevenueByDept(List<Transaction> transactions) {
        return transactions.stream().collect(
            Collectors.groupingBy(
                Transaction::department,
                Collectors.summingDouble(Transaction::amount)
            )
        );
    }

    // 2. Find Top 3 highest value transactions per department
    public Map<String, List<Transaction>> top3TransactionsPerDept(List<Transaction> transactions) {
        return transactions.stream().collect(
            Collectors.groupingBy(
                Transaction::department,
                Collectors.collectingAndThen(
                    Collectors.toList(),
                    list -> list.stream()
                                .sorted(Comparator.comparingDouble(Transaction::amount).reversed())
                                .limit(3)
                                .toList()
                )
            )
        );
    }
}
```

---

> **Next Chapter**: [12 SOLID Principles & Gang of Four (GoF) Design Patterns](12_SOLID_Principles_Design_Patterns.md)
