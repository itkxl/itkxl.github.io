## 一、基础知识
C++条件变量（condition_variable）是C++11标准库中的一种同步原语，用于在多线程程序中实现线程间的同步和互斥。其需要与一个 std::unique_lock<std::mutex> 一起使用，其中锁是用于保护条件检查的共享数据。基本的工作流程如下：
- 1、等待：线程调用conditiaon_variable的wait或wait_for方法来阻塞当前线程。在调用wai之前，线程必须锁定一个互斥锁。wait方法在等待钱会自动释放锁，并在条件变量被通知时重新获得锁。
- 2、通知：其他线程在修改了条件相关的共享数据后，可以调用notify_one或者是notify_all方法来唤醒一个或多个正在等待的线程。
## 二、使用示例
```cpp
#include <iostream>
#include <thread>
#include <mutex>
#include <condition_variable>

std::mutex mtx;
std::condition_variable cv;
bool ready = false;

void print_id(int id) {
    std::unique_lock<std::mutex> lock(mtx);
    while (!ready) { // 等待直到 `ready` 变为 `true`
        cv.wait(lock);
    }
    std::cout << "Thread " << id << '\n';
}

void go() {
    std::lock_guard<std::mutex> lock(mtx);
    ready = true;
    cv.notify_all(); // 唤醒所有线程
}

int main() {
    std::thread threads[10];
    for (int i = 0; i < 10; ++i)
        threads[i] = std::thread(print_id, i);
    std::cout << "10 threads ready to race...\n";
    go(); // 放行所有线程
    for (auto& th : threads) th.join();
    return 0;
}
```

执行main后，控制台打印如下
```bash
10 threads ready to race...
Thread 4
Thread 3
Thread 1
Thread 2
Thread 5
Thread 0
Thread 6
Thread 7
Thread 8
Thread 9
```
## 三、虚假唤醒
在上面示例代码中，在print_id方法中，判断ready状态的时候是，使用的while，而非if,这里是为什么呢？这涉及到接下来要介绍的知识点-虚假唤醒。
### 3.1 概念
夏季花心是多线程编程中使用条件变量std::conditaion_variable时可能遇到的一种现象。它指的是一个线程在没有收到适当通知的情况下，从等待状态中被唤醒。这意味着尽管条件变量出发了等待线程的唤醒，但是实际上导致唤醒的条件并非真正成立。
### 3.2 产生原因
虚假唤醒的具体原因通常是底层操作系统的线程调度和优化策略。为了提高效率和响应性，操作系统可能会在某些情况下预先唤醒线程，而不是严格等待显式的通知信号。这就要求程序在使用条件变量时必须能够正确处理虚假唤醒。
### 3.3 处理方式
可以参考示例2中代码，使用循环来处理虚假唤醒
```cpp
std::unique_lock<std::mutex> lock(mtx);
while (!ready) {  // 使用循环来处理虚假唤醒
    cv.wait(lock);  // 等待直到 `ready` 变为 `true`
}
// 进行实际的任务
std::cout << "Working...\n";
```
## 四、STL实现
### 4.1 pthread_condi_t
std::condition_variable 是 C++ 标准库中的一个类，它提供了一种跨平台的条件变量实现，允许 C++ 程序在多个操作系统上统一使用条件变量。
在 Unix-like 系统中底层实现为POSIX 线程库中的pthread_cond_t。
主要函数：
- pthread_cond_init：初始化条件变量。
- pthread_cond_wait：使线程挂起等待条件变量满足，通常与互斥锁配合使用，以防止竞态条件。
- pthread_cond_signal：唤醒至少一个等待该条件变量的线程。
- pthread_cond_broadcast：唤醒所有等待特定条件变量的线程。
- pthread_cond_destroy：销毁条件变量，释放任何相关资源。

使用条件变量时，通常配合互斥锁（pthread_mutex_t），以确保对共享资源的访问同步。
### 4.2 核心方法
```cpp
  void condition_variable::wait(unique_lock<mutex>& __lock)
  {
    _M_cond.wait(*__lock.mutex());
  }

  void condition_variable::notify_one() noexcept
  {
    _M_cond.notify_one();
  }
```
上面是贴了部分核心源码，其核心操作基本都转发给了_M_cond，而_M_cond是Mutext的内部类__condvar，其内部核心方法实现如下
```cpp
void wait(mutex& __m)
{
      int __e __attribute__((__unused__))
	= __gthread_cond_wait(&_M_cond, __m.native_handle());
      __glibcxx_assert(__e == 0);
}

notify_one() noexcept
{
      int __e __attribute__((__unused__)) = __gthread_cond_signal(&_M_cond);
      __glibcxx_assert(__e == 0);
}
```
看到这里__gthread_cond_signal&__gthread_cond_wait很明显和前面介绍的pthread_condi_t相关，最终也的确是调用到了相关方法
```cpp
static inline int
__gthread_cond_signal (__gthread_cond_t *__cond)
{
  return __gthrw_(pthread_cond_signal) (__cond);
}

static inline int
__gthread_cond_wait (__gthread_cond_t *__cond, __gthread_mutex_t *__mutex)
{
  return __gthrw_(pthread_cond_wait) (__cond, __mutex);
}
```
### 5.2 拷贝&移动
```cpp
condition_variable(const condition_variable&) = delete;
condition_variable& operator=(const condition_variable&) = delete;
```
std::condition_variable 类不能被复制或移动。不能把一个条件变量的状态从一个对象复制或移动到另一个对象。
这样的设计是为了确保条件变量的使用安全。

## 六、注意事项
- cv只能和 std::unique_lock<std::mutex>配合使用， cv_any有更高的灵活性，可以配合其他的锁进行使用。
- 尽量在循环中处理检查条件，避免虚假唤醒。