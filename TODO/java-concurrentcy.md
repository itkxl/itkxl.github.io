加锁机制 
synchronized 内置锁-互斥-可重入
关于重入：可重入意味着锁的操作颗粒度是线程，而不是调用（POSIX pthread 是调用的颗粒度）



访问共享状态的符合做操，例如 读改写 或延迟初始化 都必须使用原子操作


- 用锁来保护状态
一种常见的约定是将所有的可变状态都封装在对象内部，并通过对象的内置锁对所有访问可变状态的代码路径进行同步，使得在该对象上不会发生并发访问。
缺点是新增方法，需要记得同步对应操作。
减少锁的代码块


- 可见性
访问某个共享且可变的变量时，要对所有现成在一个锁上同步，确保可见性
加锁的含义不仅仅局限于互斥行为，还包括内存可见性，
volatile使用场景 thread while 变量
volatile只能保证可见性，无法保证原子性，一般仅用作状态标志


当且仅当满足以下条件的时候，才使用volatile
1、对变量的写入操作不依赖当前值，或者你能保证只有单个线程更新变量的值
2、该变量不会与其他状态变量一起纳入不变形条件性中
3、在访问变量时不需要加锁。


将对象发布，外部可见，也是一个危险的操作。