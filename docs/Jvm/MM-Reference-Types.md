# Java中的几种引用类型

**JDK1.2** 以前, **Java** 对引用的定义十分传统: 如果 **Reference** 类型的数据中存储的是代表另一块内存的 **起始地址**，就称该 **Reference** 为某块内存或某个对象的引用。

**JDK1.2** 之后通过引入几个类来对引用的类型进行了扩充：`java.lang.ref.*`

| Reference-Type                                         | Description          |
| ------------------------------------------------------ | -------------------- |
| [强引用(Strong-Reference)](#强引用-strong-reference)   | 保证不会回收         |
| [软引用(Soft-Reference)](#软引用-soft-reference)       | 活到 OOM 才尝试回收  |
| [弱引用(Weak-Reference)](#弱引用-weak-reference)       | 活到下次 GC          |
| [虚引用(Phantom-Reference)](#虚引用-phantom-reference) | 仅用于 GC 时获取通知 |

> 以上顺序从强 -> 弱，当有更强的引用指向对象，则服从更强引用的规则。

## 强引用-Strong-Reference

强引用就是最传统的"引用"，只要强引用关系存在，GC 永远不会回收掉被引用的对象。

_Example_:

```java
Object obj = new Object();  // "Strong-Ref" is the default reference.
```

## 软引用-Soft-Reference

软引用用来描述一些有用但非必须的对象，**只被软引用指向**的对象平常不会被 GC 回收，但在系统要发生 OOM 之前，会尝试再进行一次 GC 以腾出足够的内存空间，**只被软引用指向** 的对象会在这时候被回收。

_Example_:

```java
import java.lang.ref.SoftReference; // Since JDK 1.2

SoftReference<Object> = new SoftReference<>(new Object());
```

## 弱引用-Weak-Reference

弱引用也是用来描述非必须的对象，但强度比软引用更弱，**只被弱引用指向**的对象只能生存到下一次 GC，无论内存是否足够都会被回收。

_Example_:

```java
import java.lang.ref.WeakReference; // Since JDK 1.2

WeakReference<Object> = new WeakReference<>(new Object());
```

## 虚引用-Phantom-Reference

虚引用是最弱的一种引用，**虚引用完全不对对象的生存周期造成影响，也无法通过虚引用获得实例**。为一个对象设置虚引用的唯一目的是希望在被回收的时候能收到一个系统通知。

_Example_:

```java
import java.lang.ref.PhantomReference; // Since JDK 1.2

PhantomReference<Object> = new PhantomReference<>(new Object(), null);
```
