# Strings Deep Dive: Memory, Internals & Algorithms

> **Navigation**: [Master Index](README.md) | [Previous: Quick Revision Checklist](10_Quick_Revision_Checklist.md) | [Next: SOLID & Design Patterns](12_SOLID_Design_Principles_Patterns.md)

---

## 1. String Memory Layout & Compact Strings (Java 9+)

```
+-----------------------------------------------------------------------------------+
|                        STRING MEMORY REPRESENTATION                               |
+-----------------------------------------------------------------------------------+
|  [ Java 8 and Earlier ]                                                           |
|  String Object -> char[] value (Each char consumes 2 BYTES, even for ASCII 'A')  |
|                                                                                   |
|  [ Java 9+ Compact Strings ]                                                      |
|  String Object -> byte[] value + byte coder (LATIN1 = 0 [1 byte], UTF16 = 1 [2B]) |
|  * Saves up to 50% Heap memory in typical English/ASCII enterprise workloads!    |
+-----------------------------------------------------------------------------------+
```

---

### Q1. Deep Dive: String Constant Pool (SCP) vs. Heap Objects.
**Answer:**

```java
String s1 = "Hello";                  // Stored in String Constant Pool (SCP)
String s2 = "Hello";                  // Reuses reference from SCP (s1 == s2 is TRUE)
String s3 = new String("Hello");      // Creates a new explicit object in Heap (s1 == s3 is FALSE)
String s4 = s3.intern();              // Returns canonical reference from SCP (s1 == s4 is TRUE)
```

```
+-----------------------------------------------------------------------------------+
|                                     JAVA HEAP                                     |
|  +-----------------------------------------------------------------------------+  |
|  |                         STRING CONSTANT POOL (SCP)                          |  |
|  |  "Hello" <---------------------+ <--------------------+                     |  |
|  +--------------------------------|----------------------|---------------------+  |
|                                   |                      |                        |
|  [ Object s3: "Hello" ]           |                      |                        |
|                                   |                      |                        |
+-----------------------------------|----------------------|------------------------+
         ^                          |                      |
         |                          |                      |
     Reference s3               Reference s1           Reference s2
```

1. **Literal Allocation**: `"Hello"` checks the SCP. If present, returns reference; if absent, creates entry in SCP.
2. **`new String("Hello")`**: Allocates a new `String` instance on the regular Heap **and** ensures `"Hello"` exists in the SCP (2 objects created if literal wasn't previously in pool).
3. **`intern()`**: Checks the SCP. If string exists, returns pool reference; otherwise, adds string to pool and returns reference. *(Caution: Excessive interning of dynamic strings can cause memory pressure if pool table overflows)*.

---

## 2. String Concatenation & Performance

### Q2. Compare `String`, `StringBuilder`, and `StringBuffer`.
**Answer:**

```java
// ANTI-PATTERN: O(N^2) time and massive garbage allocation
String result = "";
for (int i = 0; i < 10_000; i++) {
    result += i; // Generates 10,000 new String objects in Heap!
}

// BEST PRACTICE: O(N) time with minimal array reallocations
StringBuilder sb = new StringBuilder(10_000); // Pre-size capacity
for (int i = 0; i < 10_000; i++) {
    sb.append(i);
}
String result = sb.toString();
```

| Dimension | `String` | `StringBuilder` | `StringBuffer` |
| :--- | :--- | :--- | :--- |
| **Mutability** | Immutable | Mutable | Mutable |
| **Thread Safety** | Thread-Safe (Immutable) | Not Thread-Safe | Thread-Safe (`synchronized` methods) |
| **Speed** | Slow for mutations | Fastest for single thread | Slower due to lock overhead |
| **Default Capacity** | Fixed | 16 characters | 16 characters |
| **Growth Formula** | N/A | `(oldCapacity * 2) + 2` | `(oldCapacity * 2) + 2` |

---

## 3. High-Frequency String Coding Challenges

### Problem 1: Longest Substring Without Repeating Characters
**Time Complexity**: $O(N)$ sliding window with single pass.
**Space Complexity**: $O(\min(N, M))$ where $M$ is alphabet size.

```java
public class LongestUniqueSubstring {
    public static int lengthOfLongestSubstring(String s) {
        if (s == null || s.isEmpty()) return 0;

        int maxLength = 0;
        int left = 0;
        Map<Character, Integer> charIndexMap = new HashMap<>();

        for (int right = 0; right < s.length(); right++) {
            char c = s.charAt(right);
            if (charIndexMap.containsKey(c)) {
                // Move left pointer past previous occurrence
                left = Math.max(left, charIndexMap.get(c) + 1);
            }
            charIndexMap.put(c, right);
            maxLength = Math.max(maxLength, right - left + 1);
        }
        return maxLength;
    }
}
```

---

### Problem 2: String Compression & Run-Length Encoding
**Question**: *"Compress string `aabcccccaaa` to `a2b1c5a3`. If compressed string is not smaller, return original."*

```java
public class StringCompressor {
    public static String compress(String str) {
        if (str == null || str.length() <= 2) return str;

        StringBuilder compressed = new StringBuilder();
        int count = 0;

        for (int i = 0; i < str.length(); i++) {
            count++;
            // If next char is different or at end of string
            if (i + 1 >= str.length() || str.charAt(i) != str.charAt(i + 1)) {
                compressed.append(str.charAt(i)).append(count);
                count = 0;
            }
        }
        return compressed.length() < str.length() ? compressed.toString() : str;
    }
}
```

---

> **Next Chapter**: [12 SOLID Design Principles & GoF Patterns](12_SOLID_Design_Principles_Patterns.md)
