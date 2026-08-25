# 02. Strings, Memory Layout & Performance Algorithms

> **Navigation**: [Master Index](README.md) | [Previous: Java Basics](01_Java_Basics_Foundations.md) | [Next: Collections & Generics](03_Collections_Framework_Generics.md)

---

## 📌 Chapter Overview
In-depth analysis of `java.lang.String` memory architecture, Compact Strings (Java 9+), String Constant Pool (SCP), `StringBuilder` vs `StringBuffer`, string concatenation bytecode mechanics, and senior coding algorithms.

---

## 1. String Memory Layout & Compact Strings (Java 9+)

### Q1. How did Java 9 Compact Strings optimize String memory footprint?
**Answer:**
Prior to Java 9, `String` stored characters using a `char[]` array, where every single character consumed **2 bytes (16 bits)** using UTF-16 encoding.

Analysis of enterprise heap dumps revealed that over **50% of heap memory** was occupied by `String` objects, and the vast majority contained only ASCII / Latin-1 characters (requiring only 1 byte per character).

#### Java 9+ Implementation:
`String` was redesigned to use a `byte[]` array combined with an encoding flag (`coder`):

```java
public final class String implements java.io.Serializable, Comparable<String>, CharSequence {
    @Stable
    private final byte[] value; // Stores 1 byte per char for Latin-1, or 2 bytes per char for UTF-16
    private final byte coder;   // 0 = LATIN1 (1 byte/char), 1 = UTF16 (2 bytes/char)
    private int hash;           // Cached 32-bit hash code
}
```

```
+-------------------------------------------------------------------------------+
|                       JAVA 8 vs JAVA 9+ STRING LAYOUT                         |
+-------------------------------------------------------------------------------+
| Java 8:  "HELLO" -> char[] value = ['H', 'E', 'L', 'L', 'O'] (10 bytes)       |
|                                                                               |
| Java 9+: "HELLO" -> byte[] value = [72, 69, 76, 76, 79]      (5 bytes)        |
|                     byte   coder = 0 (LATIN1)                 (1 byte)        |
|                     Total Memory Saved: ~40-50% across enterprise heaps!     |
+-------------------------------------------------------------------------------+
```

---

### Q2. Deep Dive: String Constant Pool (SCP) vs. Heap Objects.
**Answer:**
The **String Constant Pool (SCP)** is a dedicated caching area located inside the **Java Heap Memory** (moved from PermGen to Heap in Java 7 to allow garbage collection of unreferenced interned strings).

```java
String s1 = "Java";               // Allocated in String Constant Pool
String s2 = "Java";               // Reuses cached reference from SCP
String s3 = new String("Java");   // Creates a NEW explicit object on the Heap
String s4 = s3.intern();          // Returns the pooled reference from SCP

System.out.println(s1 == s2);     // true (Same memory address in SCP)
System.out.println(s1 == s3);     // false (s1 is in SCP, s3 is an independent Heap object)
System.out.println(s1 == s4);     // true (s4 refers to SCP reference)
```

```
       HEAP MEMORY
 +---------------------------------------------------------+
 |                                                         |
 |   +-------------------------------------------------+   |
 |   |          String Constant Pool (SCP)             |   |
 |   |                                                 |   |
 |   |   s1, s2, s4 ----> [ "Java" ] (Address 0x100)   |   |
 |   +-------------------------------------------------+   |
 |                                                         |
 |   s3 ---------------> [ "Java" ] (Address 0x500)        |
 |                       (Explicit Heap Object)            |
 +---------------------------------------------------------+
```

#### How many objects are created by `String s = new String("Hello");`?
- **2 Objects** (if `"Hello"` does not already exist in the SCP):
  1. One literal string object `"Hello"` placed inside the String Constant Pool.
  2. One explicit `String` instance allocated on the Heap referencing the pool's char array.
- **1 Object** (if `"Hello"` was already in the SCP prior to execution: only the heap object is created).

---

## 2. String Concatenation & Performance

### Q3. Compare `String`, `StringBuilder`, and `StringBuffer`.
**Answer:**

| Property | `String` | `StringBuilder` | `StringBuffer` |
| :--- | :--- | :--- | :--- |
| **Mutability** | Immutable | Mutable | Mutable |
| **Thread Safety** | Thread-safe (by immutability) | **Not thread-safe** (fastest) | **Thread-safe** (`synchronized` methods) |
| **Performance** | Slow for repeated concatenation | **Highest throughput** (Single-thread) | Slower due to lock synchronization |
| **Buffer Growth** | N/A (Creates new object) | `(oldCapacity * 2) + 2` | `(oldCapacity * 2) + 2` |
| **Introduced In** | Java 1.0 | Java 5 | Java 1.0 |

---

### Q4. How does the Java compiler optimize String concatenation (`+`)?
**Answer:**
- **Java 8**: The compiler converts string concatenation using `+` into chained `new StringBuilder().append().toString()` calls. Inside loops, this creates redundant `StringBuilder` allocations.
- **Java 9+ (JEP 280)**: String concatenation uses the `invokedynamic` bytecode instruction targeting `StringConcatFactory.makeConcatWithConstants()`. This generates optimized bytecode dynamically at runtime using method handles and pre-sized byte buffers, eliminating intermediate object allocations!

```java
// Anti-pattern inside loops:
String result = "";
for (int i = 0; i < 10_000; i++) {
    result += i; // Allocates thousands of StringBuilder instances!
}

// Production Best Practice:
StringBuilder sb = new StringBuilder(10_000); // Pre-size capacity to avoid array copying!
for (int i = 0; i < 10_000; i++) {
    sb.append(i);
}
String result = sb.toString();
```

---

## 3. High-Frequency String Coding Challenges

### Problem 1: Longest Substring Without Repeating Characters
*Given a string `s`, find the length of the longest substring without duplicate characters.*

```java
public class LongestUniqueSubstring {
    public static int lengthOfLongestSubstring(String s) {
        if (s == null || s.isEmpty()) return 0;
        
        int maxLength = 0;
        int left = 0;
        // Map character to its most recent index + 1
        Map<Character, Integer> lastSeen = new HashMap<>();

        for (int right = 0; right < s.length(); right++) {
            char c = s.charAt(right);
            if (lastSeen.containsKey(c)) {
                // Move sliding window left pointer forward past previous duplicate
                left = Math.max(left, lastSeen.get(c));
            }
            lastSeen.put(c, right + 1);
            maxLength = Math.max(maxLength, right - left + 1);
        }
        return maxLength;
    }

    public static void main(String[] args) {
        System.out.println(lengthOfLongestSubstring("abcabcbb")); // Output: 3 ("abc")
        System.out.println(lengthOfLongestSubstring("pwwkew"));    // Output: 3 ("wke")
    }
}
```
- **Time Complexity**: $O(N)$ single-pass sliding window.
- **Space Complexity**: $O(\min(N, M))$ where $M$ is the character set size.

---

### Problem 2: Group Anagrams
*Given an array of strings `strs`, group the anagrams together in any order.*

```java
public class GroupAnagramsSolution {
    public static List<List<String>> groupAnagrams(String[] strs) {
        if (strs == null || strs.length == 0) return List.of();

        Map<String, List<String>> map = new HashMap<>();
        for (String s : strs) {
            // Count frequencies of 26 lowercase English letters
            char[] count = new char[26];
            for (char c : s.toCharArray()) {
                count[c - 'a']++;
            }
            String key = String.valueOf(count); // Unique key per character frequency distribution

            map.computeIfAbsent(key, k -> new ArrayList<>()).add(s);
        }
        return new ArrayList<>(map.values());
    }

    public static void main(String[] args) {
        String[] words = {"eat", "tea", "tan", "ate", "nat", "bat"};
        System.out.println(groupAnagrams(words));
        // Output: [["bat"], ["nat", "tan"], ["ate", "eat", "tea"]]
    }
}
```
- **Time Complexity**: $O(N \cdot K)$ where $N$ is number of strings and $K$ is maximum length of a string.
- **Space Complexity**: $O(N \cdot K)$ to store grouped results.

---

### Problem 3: String Compression / Run-Length Encoding
*Compress a string in-place: `"aabcccccaaa"` $\rightarrow$ `"a2b1c5a3"`. Return original string if compressed length is not smaller.*

```java
public class StringCompression {
    public static String compress(String str) {
        if (str == null || str.length() <= 2) return str;

        StringBuilder compressed = new StringBuilder();
        int countConsecutive = 0;

        for (int i = 0; i < str.length(); i++) {
            countConsecutive++;
            // If next character is different or reached end of string
            if (i + 1 >= str.length() || str.charAt(i) != str.charAt(i + 1)) {
                compressed.append(str.charAt(i));
                compressed.append(countConsecutive);
                countConsecutive = 0;
            }
        }
        return compressed.length() < str.length() ? compressed.toString() : str;
    }

    public static void main(String[] args) {
        System.out.println(compress("aabcccccaaa")); // Output: "a2b1c5a3"
        System.out.println(compress("abcdef"));      // Output: "abcdef" (compressed is longer)
    }
}
```

---

> **Next Chapter**: [03 Collections Framework, Generics & Sequenced Collections](03_Collections_Framework_Generics.md)

