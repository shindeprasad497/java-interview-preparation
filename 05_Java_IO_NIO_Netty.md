# 05. Java I/O, NIO & Netty Architecture

> **Navigation**: [Master Index](README.md) | [Previous: Modern Java Features](04_Modern_Java_Features_8_to_21.md) | [Next: Multithreading & Concurrency](06_Multithreading_JMM_Concurrency.md)

---

## 📌 Chapter Overview
This module explores Java input/output architectures, comparing classic blocking I/O with modern **Java NIO (Non-blocking I/O)**, **Zero-Copy file transfer mechanics**, and the asynchronous **Netty EventLoop** framework.

---

## 1. Classic Blocking I/O vs. Java NIO

```
  CLASSIC I/O (Thread-Per-Connection)            JAVA NIO (Multiplexed Event Loop)
+------------------------------------+    +-----------------------------------------------+
|  Client 1 ----> [ Dedicated Thread]|    |  Client 1 ---\                                |
|  Client 2 ----> [ Dedicated Thread]|    |  Client 2 ----+--> [ Selector ] -> [ 1 Worker]|
|  Client N ----> [ Dedicated Thread]|    |  Client N ---/     (Event Loop)    [  Thread ]|
|  * High Thread Memory Overhead     |    |  * Single thread multiplexes 10,000+ sockets  |
+------------------------------------+    +-----------------------------------------------+
```

### Q1. Compare Classic I/O (BIO) vs. Java NIO (New I/O).
**Answer:**

| Feature | Classic Java I/O (`java.io`) | Java NIO (`java.nio`) |
| :--- | :--- | :--- |
| **Model** | Stream-oriented (Byte/Character streams) | **Buffer- and Channel-oriented** |
| **Blocking Mode** | Blocking (Thread stalls waiting for data) | **Non-blocking / Asynchronous** |
| **Concurrency Scaling**| Thread-per-connection (High memory overhead) | **Multiplexed via `Selector`** (Few threads handle thousands of sockets) |
| **Data Movement** | Read directly into byte arrays | Data transferred via `ByteBuffer` and `Channel` |
| **Zero-Copy Support** | No (Double buffer copying) | Yes (`FileChannel.transferTo()`) |

---

## 2. Java NIO Core Building Blocks: Channels, Buffers & Selectors

```
               [ Network Socket / File ]
                          ^
                          |  Read / Write
                          v
                   +---------------+
                   |    Channel    |
                   +---------------+
                          ^
                          |  Fills / Drains
                          v
                   +---------------+
                   |  ByteBuffer   | [ pos | limit | cap ]
                   +---------------+
                          ^
                          | Monitored By
                          v
                   +---------------+
                   |   Selector    | (OP_READ, OP_WRITE, OP_ACCEPT)
                   +---------------+
```

### Q2. Explain `ByteBuffer` state transitions (`position`, `limit`, `capacity`).
**Answer:**
- **`capacity`**: Total allocated buffer size in bytes (fixed upon creation).
- **`position`**: Next index to be written to or read from.
- **`limit`**: Index past which data cannot be read or written.

```java
ByteBuffer buffer = ByteBuffer.allocate(1024); // cap=1024, limit=1024, pos=0

// 1. Write mode: Putting data into buffer
buffer.put("Hello NIO".getBytes()); // pos moves to 9

// 2. flip(): Switch from WRITE mode to READ mode
buffer.flip(); // limit sets to old pos (9), pos resets to 0

// 3. Read mode: Reading bytes from buffer
byte[] data = new byte[buffer.remaining()];
buffer.get(data); // pos moves to limit (9)

// 4. clear(): Switch back to WRITE mode for next read cycle
buffer.clear(); // pos resets to 0, limit resets to capacity (1024)
```

---

## 3. Direct vs. Non-Direct (Heap) Buffers

### Q3. What is the difference between `ByteBuffer.allocate()` and `ByteBuffer.allocateDirect()`?
**Answer:**
- **Heap Buffer (`ByteBuffer.allocate()`)**:
  - Memory allocated inside the JVM **Java Heap**.
  - Subject to GC overhead.
  - When performing OS I/O operations, the JVM must copy the heap buffer into a temporary native OS buffer before issuing the kernel system call.
- **Direct Buffer (`ByteBuffer.allocateDirect()`)**:
  - Memory allocated in **Native OS Memory (Off-Heap)** outside JVM Garbage Collection via `malloc()`.
  - Kernel performs zero-copy socket/file I/O directly with this memory area.
  - Slower allocation/deallocation overhead, but significantly higher I/O throughput.

---

## 4. Zero-Copy File Transfer Architecture

### Q4. How does Java Zero-Copy work via `FileChannel.transferTo()`?
**Answer:**

```
 TRADITIONAL I/O (4 Context Switches + 4 Copies):
 [Disk] -> Kernel DMA Buffer -> User Space JVM Buffer -> Kernel Socket Buffer -> [NIC]
            (Context Switch)     (Context Switch)        (Context Switch)

 ZERO-COPY NIO (2 Context Switches + 0 CPU Copies):
 [Disk] -> Kernel DMA Buffer --------------------------> Network Interface Card [NIC]
                             (sendfile() syscall)
```

In traditional file-to-network transfer, data crosses user-kernel space boundaries 4 times. 

With `FileChannel.transferTo()`, the JVM utilizes the underlying Linux **`sendfile()`** system call:
- Data is transferred directly from the OS page cache to the network socket buffer by the DMA (Direct Memory Access) engine.
- Bypasses JVM memory and CPU copying completely!

```java
public void transferFileZeroCopy(File source, SocketChannel destination) throws IOException {
    try (FileChannel fileChannel = new FileInputStream(source).getChannel()) {
        long position = 0;
        long count = fileChannel.size();
        // Direct kernel-level DMA zero-copy transfer
        fileChannel.transferTo(position, count, destination);
    }
}
```

---

## 5. Netty High-Performance Networking Architecture

### Q5. Explain the Netty EventLoop and ChannelPipeline model.
**Answer:**

```
 +-------------------------------------------------------------------------------+
 |                           NETTY SERVER ARCHITECTURE                           |
 +-------------------------------------------------------------------------------+
 |                                                                               |
 |  [ BossEventLoopGroup ] (1-2 Threads)                                         |
 |    - Listens on server port for new TCP connections                           |
 |    - Accepts connection and registers SocketChannel to WorkerGroup            |
 |                                                                               |
 |  [ WorkerEventLoopGroup ] (Num Cores * 2 Threads)                             |
 |    - Reads/Writes bytes from accepted SocketChannels                          |
 |    - Executes ChannelPipeline Handlers:                                       |
 |                                                                               |
 |    [ Socket ] -> [ SslHandler ] -> [ HttpDecoder ] -> [ BusinessHandler ]    |
 |                  (Inbound Handler) (Inbound Handler)  (ChannelInboundHandler) |
 +-------------------------------------------------------------------------------+
```

1. **Boss EventLoopGroup**: Dedicated thread pool for accepting incoming connections and registering them to worker threads.
2. **Worker EventLoopGroup**: Dedicated thread pool executing non-blocking event loops for active socket channels.
3. **ChannelPipeline**: Implementation of the *Chain of Responsibility* design pattern. Intercepts inbound and outbound network data with modular handlers (decoders, encoders, authentication, business controllers).
4. **`ByteBuf`**: Netty's zero-copy, pooled replacement for `java.nio.ByteBuffer` with distinct read/write indexes (`readerIndex`, `writerIndex`), removing the need for error-prone `.flip()` calls.

---

> **Next Chapter**: [06 Multithreading, JMM & Synchronization](06_Multithreading_JMM_Concurrency.md)

