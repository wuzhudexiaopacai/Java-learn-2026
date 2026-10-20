## 1. volatile 作用是什么？

1. **保证可见性**：一个线程修改 volatile 变量，会立即刷入主内存；其他线程读取时，强制从主内存加载，不使用 CPU 缓存 / 工作内存副本。
2. **禁止指令重排序**：编译器、CPU 不能把 volatile 变量前后的指令随意调换顺序。
3. **不保证原子性**，不能解决复合操作线程安全问题（例如 `i++`）

## 2. 什么是指令重排？volatile 怎么禁止重排？

指令重排：JVM 和 CPU 为提升执行效率，在不改变单线程执行结果前提下，调整代码执行顺序。
分为：编译器重排、CPU 指令重排。
volatile 通过内存屏障（Memory Barrier）禁止重排：
- 在 volatile 写操作之后插入 `StoreStore`、`StoreLoad` 屏障：前面的读写不能重排到 volatile 写之后；volatile 写的数据刷新到主存。
- 在 volatile 读操作之前插入 `LoadLoad`、`LoadStore` 屏障：后面的读写不能重排到 volatile 读之前。
> 
> 经典案例：DCL 双重检查锁单例，不加 volatile 会发生指令重排，拿到半初始化对象。

## 3. sleep 和 wait 的区别

1. 归属：sleep 是 Thread 静态方法；wait 是 Object 方法
2. 锁：sleep **不释放锁**；wait **释放锁**
3. 使用位置：sleep 任意地方；wait 必须 synchronized 内
4. 唤醒：sleep 时间到自动醒；wait 需要 notify/notifyAll 唤醒（或超时）
5. 中断：两者被 interrupt 都会抛 InterruptedException

## 4. yield () 和 sleep ()

yield：让出 CPU，回到就绪，时间不确定，不会阻塞；
sleep：进入阻塞状态，固定休眠时间。

## 5. join 底层原理

join 底层调用 wait ()，所以 join 会释放锁；线程执行完成 JVM 自动 notifyAll。

## 6. interrupt 能不能直接停止线程？

不能！只是设置中断标记。

- 阻塞线程 (sleep/wait/join) → 抛异常，清除标记
- 运行中线程 → 标记置 true，代码需要自己判断 `isInterrupted()` 来退出

##  7.守护线程和用户线程的区别

java 线程分为用户线程与守护线程。用户线程是程序核心业务线程，只要存在运行的用户线程，JVM 就不会退出；守护线程是后台服务线程，当 JVM 中所有用户线程执行完毕，JVM 会直接退出，强制结束守护线程。守护线程需要在线程 start () 之前调用 setDaemon (true) 设置，守护线程中创建的子线程默认也是守护线程。GC 就是典型守护线程，守护线程不适合执行需要保证完成的 IO、事务等操作。

## 8.Java中wait的虚假唤醒是什么

虚假唤醒是指：**一个调用了 wait () 阻塞的线程，在没有其他线程调用 notify/notifyAll 的情况下，自己从 wait 返回了**。
这个不是 Java 或者 JVM 的 bug，底层是操作系统条件变量的规范允许的行为。
如果我们用`if`判断条件，一旦发生虚假唤醒，线程会直接往下执行业务代码，但此时等待的条件并不满足，就会出现逻辑异常。
所以 JDK 文档明确要求：**wait () 必须放在 while 循环里面判断条件**。线程被唤醒之后，会再次检查条件；条件不满足，就继续 wait，以此规避虚假唤醒。
补充一句：while 同时还能解决另一个场景：就算是正常 notify 唤醒，也可能被其他线程抢先修改了条件。

### 快速思维导图总结

- start ()：启动线程，调用 run
- run ()：线程任务
- join ()：等待别的线程执行完，底层 wait，释放锁
- sleep ()：休眠，不释放锁
- interrupt ()：打中断标记
- currentThread ()：获取当前线程
- yield ()：让出 CPU
- setDaemon ()：守护线程
- wait ()：释放锁等待（Object）
- notify/notifyAll：唤醒等待线程（Object）

## 