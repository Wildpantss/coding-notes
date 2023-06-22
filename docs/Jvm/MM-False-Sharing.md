# Java中的伪共-False-Sharing

- [Java中的伪共-False-Sharing](#java中的伪共-false-sharing)
  - [1. What-Is-False-Sharing](#1-what-is-false-sharing)
  - [2. Cache-Line](#2-cache-line)
  - [3. How-To-Avoid](#3-how-to-avoid)
    - [3.1. Classic-Solution](#31-classic-solution)
    - [3.2. Annotation](#32-annotation)

## 1. What-Is-False-Sharing

计算机缓存系统中是以 **缓存行**(*Cache-Line*) 为单位存储的, 当多线程修改互相独立的变量时, 如果这些变量共享同一个缓存行, 因为都会导致同一个缓存行失效而会无意中影响彼此的性能, 这就是**伪共享**(*False-Sharing*)。

```text
Threads                  Cache
+-----------------+      +--------+
| Thread1 / Core1 | -----|--> X   |
+-----------------+      |        |
                         |        |
+-----------------+      |        |
| Thread2 / Core2 | -----|--> Y   |
+-----------------+      +--------+

// when `X` modified, cache invalidation happens to `Y`
```

> 由于从代码中很难看出是否会出现伪共享,有人将其描述成无声的性能杀手。

## 2. Cache-Line

*Cache-Line* 是 **CPU** 和 **主存** 之间数据传输的最小单位, 缓存行通常是 **64** Byte。

当 CPU 访问某个变量时, 首先会去看 CPU-Cache 内是否有该变量, 如果有则直接从中获取, 否则就去主内存里面获取该变量, 然后把该变量所在内存区域的一个 *Cache-Line* 大小的内存复制到 Cache 中。由于存放到 Cache 行的是内存块而不是单个变量, 所以可能会把多个连续的变量存放到一个 Cache 行中。

由此, 所以通常情况下访问连续存储的数据会比随机访问要快, 访问数组结构通常比链结构快, 因为通常数组在内存中是连续分配的。

> **Note**: 一些 *JVM* 实现中大数组不是分配在连续空间的。

## 3. How-To-Avoid

为了避免由于 *False-Sharing* 导致 *Cache-Line* 从 L1 / L2 / L3 到主存之间重复载入，我们可以使用 **数据填充** 追加字节的方式来避免，即单个数据填充满一个 *Cache-Line*。

> 该方法本质上是一种 **空间换时间** 的做法。

### 3.1. Classic-Solution

在 *JDK8* 之前一般都是通过代码 **手动字节填充** 的方式来避免该问题, 也就是创建一个变量时使用填充字段填充该变量所在的缓存行, 这样就避免了将多个变量存放在同一个缓存行中。*Doug Lea* 的 [JSR166](https://jcp.org/en/jsr/detail?id=166) 中早期的 `LinkedTransferQueue` 就是使用了追加字节填充来解决伪共享, 另外在早期 `ConcurrentHashMap`、 无锁并发框架 `Disruptor` 中均使用这种技术。

```java
static final class PaddedAtomicReference<T> extends AtomicReference<T> {
  // HERE: padding objects
  // field value from parent takes 4 bytes, plus 15 padding pointers, total 64 bytes.
  Object p0, p1, p2, p3, p4, p5, p6, p7, p8, p9, pa, pb, pc, pd, pe;
  PaddedAtomicReference(T r) {
    super(r);
  }
}

public class AtomicReference<V> implements java.io.Serializable {
  private volatile V value;
  public AtomicReference(V initialValue) {
    value = initialValue;
  }
}
```

这个内部类 `PaddedAtomicReference` 相对于父类 `AtomicReference` 仅仅做了一件事情: 将共享变量追加到 64 Bytes, 它追加了15个变量共占 60 Bytes, 再加上父类的 value 变量 4 Bytes, 共 64 Bytes。

### 3.2. Annotation

`jdk.internal.vm.annotation`, since *JDK8*, 但仅在 *JDK* 内部使用。

> 可以通过以下参数在用户代码中使用, 并控制对齐(Padding)行为:
> `-XX:-RestrictContended`
> `-XX: ContendPaddingWidth`
