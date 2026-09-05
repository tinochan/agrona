# Agrona — Project Introduction

## 1. High-Level Overview

Agrona is a library of high-performance data structures and utility methods for building
low-latency, garbage-light applications in Java. It is developed and maintained by
[Real Logic Limited](https://www.real-logic.co.uk) and is the foundational toolkit behind
the [Aeron](https://github.com/aeron-io/aeron) message transport and the
[Simple Binary Encoding (SBE)](https://github.com/aeron-io/simple-binary-encoding) codec.

Its core design goals are:

- **No boxing** — data structures specialised for primitive `int`/`long` keys and values
  instead of boxing them into `Integer`/`Long`.
- **Low / zero allocation** — APIs designed so hot paths can run without producing garbage.
- **Off-heap access** — buffers that wrap `byte[]`, `ByteBuffer` (heap or direct), or a raw
  off-heap memory address, with explicit memory-ordering semantics.
- **Lock-free concurrency** — queues, ring buffers, and broadcast buffers implemented without
  `synchronized` or `ReentrantLock` for the common single-producer/single-consumer and
  many-producer/single-consumer cases.
- **Cache-friendly algorithms** — open addressing with linear probing, padded fields to avoid
  false sharing, and bit-masked power-of-two indexing.

## 2. Repository Layout

| Module                  | Purpose                                                                 |
|-------------------------|-------------------------------------------------------------------------|
| `agrona`                | The main library (`org.agrona.*`).                                      |
| `agrona-agent`          | A Java agent (`BufferAlignmentAgent`) that detects misaligned atomic buffer access. |
| `agrona-benchmarks`     | JMH benchmarks for the data structures.                                 |
| `agrona-concurrency-tests` | JCStress tests used to validate the concurrent algorithms.           |

### Package map (`agrona/src/main/java`)

- `org.agrona` — buffer interfaces/implementations, `BitUtil`, `Strings`, `AsciiEncoding`, `DeadlineTimerWheel`, `MarkFile`, io helpers.
- `org.agrona.checksum` — `Crc32`, `Crc32c` checksum implementations.
- `org.agrona.collections` — primitive-specialised maps, sets, lists, and caches.
- `org.agrona.concurrent` — queues, agents, clocks, idle strategies, id generators, errors.
- `org.agrona.concurrent.broadcast` — one-to-many broadcast buffers.
- `org.agrona.concurrent.ringbuffer` — FIFO ring buffers.
- `org.agrona.concurrent.status` — counters, positions, and status indicators.
- `org.agrona.hints` — padding/alignment hints (e.g. `CommonAlignment`).
- `org.agrona.io` — `InputStream`, `OutputStream`, and `DataInput` over direct buffers.
- `org.agrona.sbe` — helpers used by generated SBE code.

## 3. Most Used Classes and Their Key Functions

The following are the core classes you will reach for most often, with the functions you will
actually call. All signatures below reflect the current source (`2.7.0-SNAPSHOT`).

### 3.1 Buffers — `org.agrona`

**`DirectBuffer`** (read-only view interface)

- `capacity()`, `wrap(...)` — attach to a `byte[]`, `ByteBuffer`, another `DirectBuffer`, or a raw address.
- `getInt(int)`, `getLong(int)`, `getShort(int)`, `getDouble(int)`, `getByte(int)` — typed reads at a byte offset.
- `getInt(int, ByteOrder)`, `getLong(int, ByteOrder)` — byte-order-controlled reads.
- `getBytes(int, byte[])`, `getBytes(int, ByteBuffer, int)` — bulk copies.
- `getStringAscii(int)`, `getStringUtf8(int)` — decode length-prefixed strings.
- `boundsCheck(int, int)`, `addressOffset()`, `wrapAdjustment()`.

**`MutableDirectBuffer`** (read-write view interface)

- `putInt(int, int)`, `putLong(int, long)`, `putShort`, `putByte`, `putDouble` — typed writes.
- `putBytes(int, byte[])`, `putBytes(int, DirectBuffer, int, int)` — bulk copies.
- `putStringAscii(int, String)`, `putStringUtf8(int, String)` — encode length-prefixed strings (returns bytes written).
- `setMemory(int, int, byte)` — zero/fill a range.

**`UnsafeBuffer`** (default implementation, in `org.agrona.concurrent`)

- Constructors: `new UnsafeBuffer(byte[])`, `new UnsafeBuffer(ByteBuffer)`, `new UnsafeBuffer(long address, int length)`, `new UnsafeBuffer(DirectBuffer, int, int)`.
- Implements `AtomicBuffer`, so it also exposes the memory-ordered ops below.
- Effectively stateless and safe for concurrent use once wrapped (the `wrap` methods themselves are not thread-safe).

**`AtomicBuffer`** (extends `MutableDirectBuffer`)

- `getLongVolatile(int)`, `putLongVolatile(int, long)` — volatile/acquire loads and stores.
- `putLongOrdered(int, long)`, `putLongRelease(int, long)` — release stores.
- `getAndAddLong(int, long)`, `getAndAddInt`, `compareAndSetLong(int, long, long)`, `getAndSetLong`.
- Same family exists for `int`, `short`, `byte`, `char` (`getIntVolatile`, `putIntOrdered`, ...).

**`ExpandableArrayBuffer`** / **`ExpandableDirectByteBuffer`**

- `new ExpandableArrayBuffer()` (starts at 128 bytes) or `new ExpandableArrayBuffer(int)`.
- Any `put*` beyond current capacity grows the buffer automatically.

### 3.2 Ring Buffers — `org.agrona.concurrent.ringbuffer`

**`RingBuffer`** (interface) with `OneToOneRingBuffer` (single producer → single consumer)
and `ManyToOneRingBuffer` (many producers → single consumer)

- `write(int msgTypeId, DirectBuffer src, int offset, int length)` — non-blocking copy write; returns `false` if full.
- `tryClaim(int msgTypeId, int length)` / `commit(int index)` / `abort(int index)` — zero-copy claim-write-commit.
- `read(MessageHandler)` / `read(MessageHandler, int limit)` — returns the number of messages handled.
- `controlledRead(ControlledMessageHandler)` — handler can `ABORT`, `BREAK`, or `COMMIT`.
- `size()`, `capacity()`, `maxMsgLength()`, `nextCorrelationId()`, `consumerHeartbeatTime(long)`.

**`MessageHandler`** (functional interface)

- `void onMessage(int msgTypeId, MutableDirectBuffer buffer, int index, int length)`.

### 3.3 Concurrent Queues — `org.agrona.concurrent`

- `OneToOneConcurrentArrayQueue<E>` — SPSC, array backed. `offer`, `poll`, `peek`, `drain(Consumer)`, `drain(Consumer, int)`, `size`.
- `ManyToOneConcurrentArrayQueue<E>` — MPSC.
- `ManyToManyConcurrentArrayQueue<E>` — MPMC.
- `ManyToOneConcurrentLinkedQueue<E>` — MPSC, linked, unbounded.
- All are `java.util.Queue`/`BlockingQueue` compatible, lock-free, and zero-allocation on the hot path.

### 3.4 Primitive Collections — `org.agrona.collections`

- `Int2ObjectHashMap<V>`, `Int2IntHashMap`, `Int2LongHashMap`, `Long2LongHashMap`, `Object2IntHashMap`, `Object2LongHashMap`, `Long2ObjectHashMap<V>`, `Int2ObjectCache<V>`, `Object2IntCounterMap`, `IntArrayList`, `IntHashSet`.
- Open addressing with linear probing; no boxing of keys/values during iteration or `compute*`.
- Key functions (from the `Map` contract, plus): `put`, `get`, `containsKey`, `remove`, `computeIfAbsent`, `forEach((k, v) -> ...)`, `size`.

### 3.5 Agent Framework — `org.agrona.concurrent`

**`Agent`** (interface)

- `int doWork()` — return 0 when no work, positive otherwise; drives backoff.
- `String roleName()` — thread name and logging.
- `default void onStart()`, `default void onClose()` — lifecycle hooks.

**`AgentRunner`**

- Constructor: `new AgentRunner(IdleStrategy, ErrorHandler, AtomicCounter errorCounter, Agent)`.
- `AgentRunner.startOnThread(runner)` / `startOnThread(runner, threadFactory)`.
- `close()` (implements `AutoCloseable`) — stops the thread and calls the agent's `onClose()`.

Also useful: `AgentInvoker` (invoke `doWork()` on the caller's thread), `CompositeAgent`, `DynamicCompositeAgent`.

### 3.6 Idle Strategies — `org.agrona.concurrent`

**`IdleStrategy`** (interface)

- `void idle(int workCount)` — called every loop iteration with the amount of work done.
- `void idle()`, `void reset()` — used with the reset/idle pattern.

Implementations: `NoOpIdleStrategy`, `BusySpinIdleStrategy`, `YieldingIdleStrategy`, `SleepingIdleStrategy`, `SleepingMillisIdleStrategy`, `BackoffIdleStrategy`, `ControllableIdleStrategy`.

### 3.7 Clocks — `org.agrona.concurrent`

- `EpochClock.time()` → epoch millis; `NanoClock.nanoTime()` → monotonic nanos.
- Implementations: `SystemEpochClock`, `SystemNanoClock`, `CachedEpochClock`, `CachedNanoClock`, `OffsetEpochNanoClock`.
- Cached clocks trade a little accuracy for a lot of performance (they update a volatile field on a background thread).

### 3.8 Counters and Telemetry — `org.agrona.concurrent.status`

**`CountersManager`**

- Constructor: `new CountersManager(AtomicBuffer metaDataBuffer, AtomicBuffer valuesBuffer)`.
- `newCounter(String label)` → `AtomicCounter`; `allocate(String label)` → counter id; `CountersReader` for reading.

**`AtomicCounter`**

- `increment()`, `incrementOrdered()`, `decrement()`, `get()`, `getWeak()`, `set(long)`, `getAndAdd(long)` — memory-ordering variants available.

### 3.9 Broadcast — `org.agrona.concurrent.broadcast`

- `BroadcastTransmitter` — one producer writing to many consumers.
- `BroadcastReceiver`, `CopyBroadcastReceiver` — consumers; `receive(MessageHandler)`.

### 3.10 Utilities

- `BitUtil` — `align(int, int)`, `findNextPositivePowerOfTwo(int|long)`, `isPowerOfTwo`, `isAligned`, `next`, `previous`, `SIZE_OF_INT/LONG/...`.
- `Strings` — null-safe, ASCII-leaning string helpers.
- `AsciiEncoding` — encode/decode primitives to/from ASCII without `String` allocation.
- `CloseHelper` — `quietClose(AutoCloseable)`.
- `LangUtil`, `IoUtil`, `SystemUtil`, `Verify`, `MarkFile`.
- `DeadlineTimerWheel` — O(1) register/cancel timer wheel.
- `DistinctErrorLog` — deduplicated error logging (avoids filling disks).
- `SnowflakeIdGenerator` — lock-free Twitter-Snowflake-style distributed ids: `nextId()`.
- `checksum.Checksum`, `checksum.Crc32`, `checksum.Crc32c` — streaming CRC-32 / CRC-32C.
- `io.DirectBufferInputStream`, `io.DirectBufferOutputStream`, `io.DirectBufferDataInput` — JDK stream adapters over `DirectBuffer`.

## 4. Sample Use Cases

### 4.1 Typed access to heap and off-heap memory with `UnsafeBuffer`

```java
import org.agrona.concurrent.UnsafeBuffer;
import java.nio.ByteBuffer;

// Zero-copy view over a heap array
byte[] array = new byte[128];
UnsafeBuffer buffer = new UnsafeBuffer(array);

// Or a view over an off-heap direct buffer
UnsafeBuffer direct = new UnsafeBuffer(ByteBuffer.allocateDirect(128));

// Typed reads/writes at byte offsets (big-endian by default)
buffer.putLong(0, 123456789L);
buffer.putInt(8, 42);
buffer.putStringAscii(12, "hello");   // writes 4-byte length prefix + bytes
buffer.putStringAscii(21, "world");   // 12 + (4 + 5) = 21

long value = buffer.getLong(0);
int  count = buffer.getInt(8);
String msg = buffer.getStringAscii(12); // length prefix read automatically
```

### 4.2 Auto-growing buffer

```java
import org.agrona.ExpandableArrayBuffer;

ExpandableArrayBuffer buf = new ExpandableArrayBuffer(); // 128 bytes initial
buf.putLong(1000, 1L); // grows the backing array as needed
buf.putStringUtf8(8, "grows on demand");
```

### 4.3 Memory-ordered (atomic) access

```java
import org.agrona.concurrent.UnsafeBuffer;
import java.nio.ByteBuffer;

UnsafeBuffer shared = new UnsafeBuffer(ByteBuffer.allocateDirect(64));

shared.putLongOrdered(0, 1);            // release store
long v = shared.getLongVolatile(0);     // acquire load
long prev = shared.getAndAddLong(8, 1); // atomic increment
boolean ok = shared.compareAndSetLong(16, 10, 11);
```

### 4.4 Ring buffer — one producer to one consumer

```java
import org.agrona.MutableDirectBuffer;
import org.agrona.concurrent.UnsafeBuffer;
import org.agrona.concurrent.ringbuffer.OneToOneRingBuffer;
import org.agrona.concurrent.ringbuffer.RingBufferDescriptor;
import java.nio.ByteBuffer;

int dataCapacity = 1024; // must be a power of two
UnsafeBuffer storage = new UnsafeBuffer(
    ByteBuffer.allocateDirect(dataCapacity + RingBufferDescriptor.TRAILER_LENGTH));
OneToOneRingBuffer ring = new OneToOneRingBuffer(storage);

// Producer
MutableDirectBuffer msg = new UnsafeBuffer(ByteBuffer.allocateDirect(32));
msg.putInt(0, 7);
msg.putStringAscii(4, "ping");
boolean written = ring.write(1 /* msgTypeId */, msg, 0, 32);

// Consumer (returns the number of messages handled)
int messages = ring.read((msgTypeId, buffer, index, length) ->
    System.out.println("type=" + msgTypeId + ", value=" + buffer.getInt(index)));
```

### 4.5 Zero-copy claim / commit / abort

```java
int index = ring.tryClaim(2, 8); // claim 8 bytes without copying
if (index > 0)
{
    try
    {
        ring.buffer().putLong(index, 42L);
        ring.commit(index);
    }
    catch (final Exception ex)
    {
        ring.abort(index); // turn the claim into padding, consumer can proceed
    }
}
```

### 4.6 Concurrent queue with draining

```java
import org.agrona.concurrent.OneToOneConcurrentArrayQueue;

OneToOneConcurrentArrayQueue<String> queue = new OneToOneConcurrentArrayQueue<>(1024);

queue.offer("a");
queue.offer("b");

String head = queue.poll();          // "a", null if empty
int drained = queue.drain(s -> System.out.println("got " + s)); // consumes the rest
```

### 4.7 Primitive-specialised map (no boxing)

```java
import org.agrona.collections.Int2ObjectHashMap;

Int2ObjectHashMap<String> map = new Int2ObjectHashMap<>();
map.put(1, "one");
map.put(2, "two");

String v = map.get(1);
map.forEach((key, val) -> System.out.println(key + "=" + val)); // keys not boxed
```

### 4.8 Agent framework with an idle strategy

```java
import org.agrona.concurrent.Agent;
import org.agrona.concurrent.AgentRunner;
import org.agrona.concurrent.YieldingIdleStrategy;

Agent consumer = new Agent()
{
    public int doWork()
    {
        return ring.read((msgTypeId, buffer, index, length) ->
            System.out.println("got " + buffer.getStringAscii(index)));
    }

    public String roleName()
    {
        return "consumer";
    }
};

AgentRunner runner = new AgentRunner(
    new YieldingIdleStrategy(), // idle when doWork() returns 0
    Throwable::printStackTrace, // error handler
    null,                       // optional AtomicCounter for error counts
    consumer);

Thread thread = AgentRunner.startOnThread(runner);
// ... later:
runner.close(); // stops the agent thread and calls consumer.onClose()
```

### 4.9 Counters for telemetry / position tracking

```java
import org.agrona.concurrent.UnsafeBuffer;
import org.agrona.concurrent.status.AtomicCounter;
import org.agrona.concurrent.status.CountersManager;
import java.nio.ByteBuffer;

CountersManager counters = new CountersManager(
    new UnsafeBuffer(ByteBuffer.allocateDirect(8 * 1024)),  // metadata
    new UnsafeBuffer(ByteBuffer.allocateDirect(64 * 1024))); // values

AtomicCounter received = counters.newCounter("messages received");
received.increment();
received.incrementOrdered();
long total = received.get();
```

### 4.10 Common utilities

```java
import org.agrona.BitUtil;

int size = BitUtil.findNextPositivePowerOfTwo(1000); // 1024
int aligned = BitUtil.align(14, 8);                  // 16
boolean powerOfTwo = BitUtil.isPowerOfTwo(1024);     // true
```

## 5. Related Projects and Next Steps

- **Aeron** — high-performance messaging built on Agrona's buffers, ring buffers, agents, and counters.
- **SBE (Simple Binary Encoding)** — a codec that emits Agrona `MutableDirectBuffer` based decoders/encoders.
- Full API reference: [Agrona Javadocs](https://www.javadoc.io/doc/org.agrona/agrona).
- Release notes: `CHANGELOG.adoc` in this repository.
