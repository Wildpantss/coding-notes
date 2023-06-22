# Compare-And-Swap-CAS

*CAS* 是一种 **原子操作**, 用于避免多线程同时操作某一数据时导致的不一致问题。

用伪 *C* 代码来描述 *CAS* 的过程:

```c
/* Returns 1 when success, 0 when failed. */
int cas(long *addr, long old, long new) {
  /* Executes ATOMICALLY. */
  if (*addr != old) { return 0; }
  else { *addr = new; return 1; }
}
```

> 不要混淆 [**"CAS自旋锁"**](#cas自旋锁) 和 **"CAS操作"** 本身。
>
> *CAS* 操作通过硬件确保原子性, 不是软件能够简单实现的。(可以发现 `sum.misc.unsafe` 中相关代码是 `native` 的)

- [Compare-And-Swap-CAS](#compare-and-swap-cas)
  - [CAS自旋锁](#cas自旋锁)
  - [CAS自旋实现原子操作的局限性](#cas自旋实现原子操作的局限性)
    - [ABA问题](#aba问题)
    - [反复失败时开销大](#反复失败时开销大)
    - [只能保证一个共享变量操作的原子性](#只能保证一个共享变量操作的原子性)

## CAS自旋锁

***CAS*自旋锁(*CAS-Spin-Lock*)** 是一种 **轻量级** 的同步机制, 因为不需要挂起线程, **减少了 *OS* 调度的上下文切换开销**。

使用 *CAS* 来保证一段操作的原子性, 大体流程如下:

1. 保存旧值
2. 基于旧值做一系列操作得到新值(也就是这一段需要保证原子性)
3. 使用 CAS 尝试将新值写入内存
   - 如果成功则结束
   - 如果失败, 则回到 2 循环

如上逻辑的代码示例:

```java
while (true) {
  int oldVal = target;        // (1) temp the old value
  int newVal = calc(oldVal);  // (2) calculate base on old value

  // (3) try CAS
  boolean success = target.compareAndSet(target, newVal);
  if (!success) continue;     // re-try calc and set when CAS failed
  if (success) break;         // done when CAS success
}
```

> *CAS* 自旋锁是一种 **乐观锁**, 适合在资源大多数情况下只被一个线程持有的情况下使用, 否则自旋会浪费大量 *CPU* 时间。

## CAS自旋实现原子操作的局限性

### ABA问题

*CAS* 在操作值的时候仅仅检查是否发生变化, 但如果在当前线程操作的同时, 其他线程将观察对象的值从 *A* 修改到了 *B* 又修改回了 *A*, *CAS* 操作将无法发现其发生过变化。

如何解决? 使用版本号!

每次更新的时候标记一下版本号, `A -> B -> A` 就变成: `1A -> 2B -> 3A`, *CAS* 就能够判断变化了。

对应 `java.util.concurrent.atomic.AtomicStampedReference`

### 反复失败时开销大

参照 [***CAS*自旋锁**](#cas自旋锁) 的逻辑, 如果写入失败则需要保证原子性的计算步骤需要全部重新计算。因此如果 *CAS* 操作反复失败,
虽然节省了 OS 上下文切换的开销, 但 ***CPU* 资源将被大幅浪费**, 反而得不偿失。

### 只能保证一个共享变量操作的原子性

受 *CAS* 操作本身的特性所限制, **只能保证对一个共享变量操作的原子性**。

但是可以将多个共享变量合并成一个共享对象, 这样就可以借助 *CAS* 来完成原子性保证了。

对应 `java.util.concurrent.atomic.AtomicReference`
