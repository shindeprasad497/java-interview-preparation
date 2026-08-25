# 11. Coding, Data Structures & Concurrency Challenges

> **Navigation**: [Master Index](README.md) | [Previous: JVM Troubleshooting](10_JVM_Troubleshooting_Dumps_Profiling.md) | [Next: SOLID Principles & Design Patterns](12_SOLID_Principles_Design_Patterns.md)

---

## 📌 Chapter Overview
This module contains high-frequency production coding challenges frequently tested in **Senior & Lead Java Technical Coding Rounds**, including Custom BlockingQueues, Thread-Safe Singletons, LRU Caches, and Stream transformations.

---

## 1. Thread-Safe Singleton Patterns

### Problem 1: Implement a 100% Thread-Safe Singleton.
**Answer:**

#### Approach A: Double-Checked Locking (DCL) with `volatile`
```java
public class DclSingleton {
    // volatile is MANDATORY to prevent instruction reordering during object instantiation!
    private static volatile DclSingleton instance;

    private DclSingleton() {
        if (instance != null) {
            throw new RuntimeException("Use getInstance() to instantiate!");
        }
    }

    public static DclSingleton getInstance() {
        if (instance == null) { // 1st Check (No locking overhead)
            synchronized (DclSingleton.class) {
                if (instance == null) { // 2nd Check (Inside lock)
                    instance = new DclSingleton();
                }
            }
        }
        return instance;
    }
}
```

#### Approach B: Bill Pugh Singleton (Initialization-on-demand Holder Idiom - Recommended)
```java
public class BillPughSingleton {
    private BillPughSingleton() {}

    // Static nested class is NOT loaded until getInstance() is invoked!
    private static class InstanceHolder {
        private static final BillPughSingleton INSTANCE = new BillPughSingleton();
    }

    public static BillPughSingleton getInstance() {
        return InstanceHolder.INSTANCE; // ClassLoader guarantees thread safety with zero locks!
    }
}
```

---

## 2. Custom Bounded `BlockingQueue` from Scratch

### Problem 2: Implement a Thread-Safe Bounded BlockingQueue.
*Requirements*: Support `put(item)` (blocks if full) and `take()` (blocks if empty) using `ReentrantLock` and `Condition`.

```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public class CustomBoundedBlockingQueue<T> {
    private final Object[] items;
    private int count = 0;
    private int putIndex = 0;
    private int takeIndex = 0;

    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();

    public CustomBoundedBlockingQueue(int capacity) {
        if (capacity <= 0) throw new IllegalArgumentException();
        this.items = new Object[capacity];
    }

    public void put(T item) throws InterruptedException {
        lock.lock();
        try {
            // Guard against spurious wakeups using a while loop!
            while (count == items.length) {
                notFull.await(); // Blocks until space becomes available
            }

            items[putIndex] = item;
            if (++putIndex == items.length) putIndex = 0;
            count++;

            notEmpty.signal(); // Notify waiting consumers
        } finally {
            lock.unlock();
        }
    }

    @SuppressWarnings("unchecked")
    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0) {
                notEmpty.await(); // Blocks until an item is available
            }

            T item = (T) items[takeIndex];
            items[takeIndex] = null; // Prevent memory leak
            if (++takeIndex == items.length) takeIndex = 0;
            count--;

            notFull.signal(); // Notify waiting producers
            return item;
        } finally {
            lock.unlock();
        }
    }
}
```

---

## 3. Custom LRU Cache using Doubly Linked List + HashMap

```java
import java.util.HashMap;
import java.util.Map;

public class CustomLRUCache<K, V> {
    private static class Node<K, V> {
        K key;
        V value;
        Node<K, V> prev, next;
        Node(K key, V value) { this.key = key; this.value = value; }
    }

    private final int capacity;
    private final Map<K, Node<K, V>> map = new HashMap<>();
    private final Node<K, V> head = new Node<>(null, null); // Dummy head
    private final Node<K, V> tail = new Node<>(null, null); // Dummy tail

    public CustomLRUCache(int capacity) {
        this.capacity = capacity;
        head.next = tail;
        tail.prev = head;
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
                Node<K, V> removed = removeTail();
                map.remove(removed.key);
            }
            Node<K, V> newNode = new Node<>(key, value);
            map.put(key, newNode);
            addToHead(newNode);
        }
    }

    private void addToHead(Node<K, V> node) {
        node.next = head.next;
        node.prev = head;
        head.next.prev = node;
        head.next = node;
    }

    private void removeNode(Node<K, V> node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }

    private void moveToHead(Node<K, V> node) {
        removeNode(node);
        addToHead(node);
    }

    private Node<K, V> removeTail() {
        Node<K, V> res = tail.prev;
        removeNode(res);
        return res;
    }
}
```

---

## 4. Complex Stream API Data Transformations

### Problem 4: Grouping, Partitioning & Top-N Analysis
```java
public record Employee(String name, String department, double salary, int age) {}

public class StreamExercises {
    
    // 1. Find Highest-Paid Employee per Department
    public Map<String, Optional<Employee>> topEarnerPerDept(List<Employee> employees) {
        return employees.stream().collect(
            Collectors.groupingBy(
                Employee::department,
                Collectors.maxBy(Comparator.comparingDouble(Employee::salary))
            )
        );
    }

    // 2. Calculate Average Salary per Department
    public Map<String, Double> averageSalaryPerDept(List<Employee> employees) {
        return employees.stream().collect(
            Collectors.groupingBy(
                Employee::department,
                Collectors.averagingDouble(Employee::salary)
            )
        );
    }

    // 3. Character Frequency Count in String
    public Map<Character, Long> charFrequency(String input) {
        return input.chars()
                    .mapToObj(c -> (char) c)
                    .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()));
    }
}
```

---

> **Next Chapter**: [12 SOLID Principles & Gang of Four (GoF) Design Patterns](12_SOLID_Principles_Design_Patterns.md)

