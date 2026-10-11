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
