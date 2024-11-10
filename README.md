
`CountDownLatch` 是 Java 并发包（`java.util.concurrent`）中的一个同步辅助类，它允许一个或多个线程等待一组操作完成。


### 一、设计理念


`CountDownLatch` 是基于 AQS（AbstractQueuedSynchronizer）实现的。其核心思想是**维护一个倒计数**，每次倒计数减少到零时，等待的线程才会继续执行。它的主要设计目标是允许多个线程协调完成一组任务。


#### 1\. 构造函数与计数器



```


|  | public CountDownLatch(int count) { |
| --- | --- |
|  | if (count < 0) throw new IllegalArgumentException("count < 0"); |
|  | this.sync = new Sync(count); |
|  | } |


```

构造 `CountDownLatch` 时传入的 `count` 决定了计数器的初始值。该计数器控制了线程的释放。


#### 2\. AQS 支持的核心操作


AQS 是 `CountDownLatch` 的基础，通过自定义内部类 `Sync` 实现，`Sync` 继承了 AQS 并提供了必要的方法。以下是关键操作：


* `acquireShared(int arg)`: 如果计数器值为零，表示所有任务已完成，线程将获得许可。
* `releaseShared(int arg)`: 每次调用 `countDown()`，会减少计数器，当计数器降到零时，AQS 将释放所有等待的线程。


#### 3\. 实现细节


* `countDown()`：调用 `releaseShared()` 减少计数器，并通知等待线程。
* `await()`：调用 `acquireSharedInterruptibly(1)`，如果计数器非零则阻塞等待。


### 二、底层原理


`CountDownLatch` 的核心是基于 `AbstractQueuedSynchronizer`（AQS）来管理计数器状态的。AQS 是 JUC 中许多同步工具的基础，通过一个独占/共享模式的同步队列实现线程的管理和调度。`CountDownLatch` 采用 AQS 的**共享锁机制**来控制多个线程等待一个条件。


#### 1\. AQS 的共享模式


AQS 设计了两种同步模式：**独占模式**（exclusive）和**共享模式**（shared）。`CountDownLatch` 使用共享模式：


* **独占模式**：每次只能一个线程持有锁，如 `ReentrantLock`。
* **共享模式**：允许多个线程共享锁状态，如 `Semaphore` 和 `CountDownLatch`。


`CountDownLatch` 的 `await()` 和 `countDown()` 方法对应于 AQS 的 `acquireShared()` 和 `releaseShared()` 操作。`acquireShared()` 会检查同步状态（计数器值），若状态为零则立即返回，否则阻塞当前线程，进入等待队列。`releaseShared()` 用于减少计数器并唤醒所有等待线程。


#### 2\. Sync 内部类的设计


`CountDownLatch` 通过一个私有的内部类 `Sync` 来实现同步逻辑。`Sync` 继承自 `AQS`，并重写 `tryAcquireShared(int arg)` 和 `tryReleaseShared(int arg)` 方法。



```


|  | static final class Sync extends AbstractQueuedSynchronizer { |
| --- | --- |
|  | Sync(int count) { |
|  | setState(count); |
|  | } |
|  |  |
|  | protected int tryAcquireShared(int acquires) { |
|  | return (getState() == 0) ? 1 : -1; |
|  | } |
|  |  |
|  | protected boolean tryReleaseShared(int releases) { |
|  | // 自旋减计数器 |
|  | for (;;) { |
|  | int c = getState(); |
|  | if (c == 0) |
|  | return false; |
|  | int nextc = c - 1; |
|  | if (compareAndSetState(c, nextc)) |
|  | return nextc == 0; |
|  | } |
|  | } |
|  | } |


```

* **tryAcquireShared(int)**：当计数器为零时返回 1（成功获取锁），否则返回 \-1（阻塞）。
* **tryReleaseShared(int)**：每次 `countDown()` 减少计数器值，当计数器到达零时返回 `true`，唤醒所有阻塞线程。


#### 3\. CAS 操作确保线程安全


`tryReleaseShared` 方法使用 CAS（compare\-and\-set）更新计数器，避免了锁的开销。CAS 操作由 CPU 原语（如 `cmpxchg` 指令）支持，实现了高效的非阻塞操作。这种设计保证了 `countDown()` 的线程安全性，使得多个线程能够并发地减少计数器。


#### 4\. 内部的 ConditionObject


`CountDownLatch` 不支持复用，因为 AQS 的 `ConditionObject` 被设计为单一触发模式。计数器一旦降至零，`CountDownLatch` 无法重置，只能释放所有线程，而不能再次设置初始计数器值。这就是其不可复用的根本原因。


### 三、应用场景


1. **等待多线程任务完成**：`CountDownLatch` 常用于需要等待一组线程完成其任务后再继续的场景，如批处理任务。
2. **并行执行再汇总**：在某些数据分析或计算密集型任务中，将任务分割成多个子任务并行执行，主线程等待所有子任务完成后再汇总结果。
3. **多服务依赖协调**：当一个服务依赖多个其他服务时，可以使用 `CountDownLatch` 来同步各个服务的调用，并确保所有依赖服务准备好之后再执行主任务。


### 四、示例代码


以下示例展示如何使用 `CountDownLatch` 实现一个并发任务等待所有子任务完成的机制。



```


|  | import java.util.concurrent.CountDownLatch; |
| --- | --- |
|  |  |
|  | public class CountDownLatchExample { |
|  | private static final int TASK_COUNT = 5; |
|  | private static CountDownLatch latch = new CountDownLatch(TASK_COUNT); |
|  |  |
|  | public static void main(String[] args) throws InterruptedException { |
|  | for (int i = 0; i < TASK_COUNT; i++) { |
|  | new Thread(new Task(i + 1, latch)).start(); |
|  | } |
|  |  |
|  | // 主线程等待所有任务完成 |
|  | latch.await(); |
|  | System.out.println("所有任务已完成，继续主线程任务"); |
|  | } |
|  |  |
|  | static class Task implements Runnable { |
|  | private final int taskNumber; |
|  | private final CountDownLatch latch; |
|  |  |
|  | Task(int taskNumber, CountDownLatch latch) { |
|  | this.taskNumber = taskNumber; |
|  | this.latch = latch; |
|  | } |
|  |  |
|  | @Override |
|  | public void run() { |
|  | try { |
|  | System.out.println("子任务 " + taskNumber + " 开始执行"); |
|  | Thread.sleep((int) (Math.random() * 1000)); // 模拟任务执行时间 |
|  | System.out.println("子任务 " + taskNumber + " 完成"); |
|  | } catch (InterruptedException e) { |
|  | Thread.currentThread().interrupt(); |
|  | } finally { |
|  | latch.countDown(); // 完成一个任务，计数器减一 |
|  | } |
|  | } |
|  | } |
|  | } |


```

### 五、与其他同步工具的对比


#### 1\. CyclicBarrier


**原理和用途**：


* `CyclicBarrier` 也允许一组线程相互等待，直到所有线程到达屏障位置（barrier point）。
* 它适合用于**多阶段任务**或**分阶段汇聚**，如处理分块计算时每阶段汇总结果。


**底层实现**：


* `CyclicBarrier` 内部通过 `ReentrantLock` 和 `Condition` 实现，屏障次数可以重置，从而支持循环使用。


**与 CountDownLatch 的对比**：


* `CyclicBarrier` 的**可复用性**使其适合重复的同步场景，而 `CountDownLatch` 是一次性的。
* `CountDownLatch` 更灵活，允许任意线程调用 `countDown()`，适合分布式任务。`CyclicBarrier` 需要指定的线程达到屏障。


#### 2\. Semaphore


**原理和用途**：


* `Semaphore` 主要用于**控制资源访问**的并发数量，如限制数据库连接池的访问。


**底层实现**：


* `Semaphore` 基于 AQS 的共享模式实现，类似于 `CountDownLatch`，但允许通过指定的“许可证”数量控制资源。


**与 CountDownLatch 的对比**：


* `Semaphore` 可以动态增加/减少许可，而 `CountDownLatch` 只能递减。
* `Semaphore` 适合控制访问限制，而 `CountDownLatch` 用于同步点倒计数。


#### 3\. Phaser


**原理和用途**：


* `Phaser` 是 `CyclicBarrier` 的增强版，允许动态调整参与线程的数量。
* 适合多阶段任务同步，并能随时增加或减少参与线程。


**底层实现**：


* `Phaser` 内部包含一个计数器，用于管理当前阶段的参与线程，允许任务动态注册或注销。


**与 CountDownLatch 的对比**：


* `Phaser` 更适合复杂场景，能够灵活控制阶段和参与线程；`CountDownLatch` 的结构简单，只能用于一次性同步。
* `Phaser` 的设计更复杂，适合长时间、多线程协调任务，而 `CountDownLatch` 更适合简单任务等待。


#### 4、总结


`CountDownLatch` 是一个轻量级、不可复用的倒计数同步器，适合简单的一次性线程协调。其基于 AQS 的共享锁实现使得线程等待和计数器更新具有高效的并发性。虽然 `CountDownLatch` 不具备重用性，但其设计简洁，尤其适合需要等待多线程任务完成的场景。


与其他 JUC 工具相比：


* `CyclicBarrier` 更适合多阶段同步、阶段性汇总任务。
* `Semaphore` 适合资源访问控制，具有可控的许可量。
* `Phaser` 灵活性更高，适合动态参与线程、复杂多阶段任务。


选择适合的同步工具，取决于任务的性质、线程参与动态性以及是否需要重用同步控制。


 本博客参考[FlowerCloud机场](https://hanlianfangzhi.com)。转载请注明出处！
