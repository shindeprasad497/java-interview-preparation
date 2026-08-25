# Java Coding & Problem Solving: Production Patterns & Interview Challenges

> **Navigation**: [Master Index](README.md) | [Previous: Collections & Concurrency](02_Collections_Generics_Concurrency.md) | [Next: Spring Core & Boot Internals](04_Spring_Core_Boot_Internals.md)

---

## 1. Concurrency Coding Patterns

### Problem 1: Thread-Safe Singleton Implementation
**Question**: *"Implement a 100% thread-safe Singleton in Java. Explain why `volatile` is required in Double-Checked Locking."*

#### Solution A: Double-Checked Locking (DCL)
```java
public class ThreadSafeDCLSingleton {
    // volatile is MANDATORY to prevent CPU instruction reordering during instantiation
    private static volatile ThreadSafeDCLSingleton instance;

    private ThreadSafeDCLSingleton() {
        // Guard against reflection attacks
        if (instance != null) {
            throw new IllegalStateException("Instance already exists!");
        }
    }

    public static ThreadSafeDCLSingleton getInstance() {
        if (instance == null) { // 1st Check (No lock overhead for existing instances)
            synchronized (ThreadSafeDCLSingleton.class) {
                if (instance == null) { // 2nd Check (Guarantees single instantiation)
                    // Instantiation bytecode involves 3 steps:
                    // 1. Allocate memory -> 2. Initialize constructor -> 3. Assign memory address to reference
                    // Without 'volatile', JVM could reorder steps 2 and 3, exposing a half-initialized object!
                    instance = new ThreadSafeDCLSingleton();
                }
            }
        }
        return instance;
    }
}
```

#### Solution B: Bill Pugh Singleton Holder (Eager Loading via ClassLoader)
```java
public class BillPughSingleton {
    private BillPughSingleton() {}

    // Static nested class is only loaded into memory when getInstance() is called
    private static class Holder {
        private static final BillPughSingleton INSTANCE = new BillPughSingleton();
    }

    public static BillPughSingleton getInstance() {
        return Holder.INSTANCE;
    }
}
```

---

### Problem 2: Producer-Consumer Pattern with Custom `Condition` Variables
**Question**: *"Implement a bounded thread-safe queue from scratch using `ReentrantLock` and `Condition` variables (do not use `ArrayBlockingQueue`)."*

```java
import java.util.LinkedList;
import java.util.Queue;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public class BoundedBlockingQueue<T> {
    private final Queue<T> queue = new LinkedList<>();
    private final int capacity;
    private final ReentrantLock lock = new ReentrantLock(true); // Fair lock
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();

    public BoundedBlockingQueue(int capacity) {
        this.capacity = capacity;
    }

    public void put(T item) throws InterruptedException {
        lock.lock();
        try {
            while (queue.size() == capacity) {
                notFull.await(); // Releases lock and suspends producer thread
            }
            queue.add(item);
            notEmpty.signal(); // Wakes up one waiting consumer thread
        } finally {
            lock.unlock();
        }
    }

    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (queue.isEmpty()) {
                notEmpty.await(); // Releases lock and suspends consumer thread
            }
            T item = queue.poll();
            notFull.signal(); // Wakes up one waiting producer thread
            return item;
        } finally {
            lock.unlock();
        }
    }

    public int size() {
        lock.lock();
        try {
            return queue.size();
        } finally {
            lock.unlock();
        }
    }
}
```

---

### Problem 3: Custom Thread-Safe LRU Cache with $O(1)$ Operations
**Question**: *"Implement an LRU Cache from scratch with $O(1)$ `get` and `put` methods without inheriting from `LinkedHashMap`."*

```java
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.locks.ReentrantLock;

public class CustomLRUCache<K, V> {
    private static class Node<K, V> {
        K key;
        V value;
        Node<K, V> prev, next;
        Node(K key, V value) { this.key = key; this.value = value; }
    }

    private final int capacity;
    private final ConcurrentHashMap<K, Node<K, V>> map = new ConcurrentHashMap<>();
    private final ReentrantLock lock = new ReentrantLock();
    private final Node<K, V> head = new Node<>(null, null); // Dummy head
    private final Node<K, V> tail = new Node<>(null, null); // Dummy tail

    public CustomLRUCache(int capacity) {
        this.capacity = capacity;
        head.next = tail;
        tail.prev = head;
    }

    public V get(K key) {
        lock.lock();
        try {
            Node<K, V> node = map.get(key);
            if (node == null) return null;
            moveToHead(node); // Mark as most recently used
            return node.value;
        } finally {
            lock.unlock();
        }
    }

    public void put(K key, V value) {
        lock.lock();
        try {
            Node<K, V> node = map.get(key);
            if (node != null) {
                node.value = value;
                moveToHead(node);
            } else {
                if (map.size() >= capacity) {
                    Node<K, V> lru = removeTail();
                    if (lru != null) map.remove(lru.key);
                }
                Node<K, V> newNode = new Node<>(key, value);
                addNode(newNode);
                map.put(key, newNode);
            }
        } finally {
            lock.unlock();
        }
    }

    private void addNode(Node<K, V> node) {
        node.prev = head;
        node.next = head.next;
        head.next.prev = node;
        head.next = node;
    }

    private void removeNode(Node<K, V> node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }

    private void moveToHead(Node<K, V> node) {
        removeNode(node);
        addNode(node);
    }

    private Node<K, V> removeTail() {
        Node<K, V> lru = tail.prev;
        if (lru == head) return null;
        removeNode(lru);
        return lru;
    }
}
```

---

## 2. Stream API & Functional Transformations

### Problem 4: Complex Data Aggregations with Streams
**Question**: *"Given a list of transactions, find the top 3 highest-spending customers by country and calculate their total spend."*

```java
public record Transaction(String txId, String customerId, String country, BigDecimal amount) {}
public record CustomerSpend(String customerId, BigDecimal totalSpend) {}

public class StreamAggregator {
    public static Map<String, List<CustomerSpend>> top3CustomersByCountry(List<Transaction> transactions) {
        return transactions.stream()
            // 1. Group by Country -> Customer -> Sum amounts
            .collect(Collectors.groupingBy(
                Transaction::country,
                Collectors.groupingBy(
                    Transaction::customerId,
                    Collectors.reducing(
                        BigDecimal.ZERO,
                        Transaction::amount,
                        BigDecimal::add
                    )
                )
            ))
            // 2. Transform inner map into sorted top 3 list
            .entrySet().stream()
            .collect(Collectors.toMap(
                Map.Entry::getKey,
                entry -> entry.getValue().entrySet().stream()
                    .map(e -> new CustomerSpend(e.getKey(), e.getValue()))
                    .sorted(Comparator.comparing(CustomerSpend::totalSpend).reversed())
                    .limit(3)
                    .toList()
            ));
    }
}
```

---

### Problem 5: Frequency Count & Anagram Grouping
**Question**: *"Given an array of strings, group anagrams together in $O(N \cdot K)$ time complexity."*

```java
public class AnagramGrouper {
    public static List<List<String>> groupAnagrams(String[] words) {
        if (words == null || words.length == 0) return List.of();

        return new ArrayList<>(Arrays.stream(words)
            .collect(Collectors.groupingBy(word -> {
                // Character frequency hash key "#1#0#2..."
                int[] count = new int[26];
                for (char c : word.toCharArray()) count[c - 'a']++;
                return Arrays.toString(count);
            }))
            .values());
    }
}
```

---

## 3. Production Incident Coding Challenge

### Problem 6: Programmatic Deadlock Detection & Logging
**Question**: *"Write a utility background thread that detects deadlocks in a running Java service and logs the full stack trace of all locked threads."*

```java
import java.lang.management.ManagementFactory;
import java.lang.management.ThreadInfo;
import java.lang.management.ThreadMXBean;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class DeadlockDetector {
    private static final Logger log = LoggerFactory.getLogger(DeadlockDetector.class);

    public static void startMonitoring() {
        var scheduler = Executors.newSingleThreadScheduledExecutor(r -> {
            Thread t = new Thread(r, "deadlock-detector");
            t.setDaemon(true);
            return t;
        });

        scheduler.scheduleAtFixedRate(() -> {
            ThreadMXBean bean = ManagementFactory.getThreadMXBean();
            long[] deadlockedThreads = bean.findDeadlockedThreads();

            if (deadlockedThreads != null && deadlockedThreads.length > 0) {
                log.error("CRITICAL: DEADLOCK DETECTED! Total threads involved: {}", deadlockedThreads.length);
                ThreadInfo[] threadInfos = bean.getThreadInfo(deadlockedThreads, true, true);
                for (ThreadInfo ti : threadInfos) {
                    log.error("Thread '{}' [ID: {}] is BLOCKED on lock '{}' held by thread '{}'",
                        ti.getThreadName(), ti.getThreadId(), ti.getLockName(), ti.getLockOwnerName());
                    for (StackTraceElement ste : ti.getStackTrace()) {
                        log.error("    at {}", ste);
                    }
                }
            }
        }, 10, 30, TimeUnit.SECONDS);
    }
}
```

---

> **Next Chapter**: [04 Spring Core & Boot Internals](04_Spring_Core_Boot_Internals.md)
