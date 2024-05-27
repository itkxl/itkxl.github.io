## 一、背景
日常在进行JNI开发中，尤其是在多线程的环境下，数据的管理和处理非常重要。其中队列作为基础数据结构，应用广泛，特别是在生产者-消费者问题中扮演着核心角色。然而，普通队列在并发环境下存在线程安全问题，这就需要专门的同步机制来保证数据的一致性和完整性。

期初接触到线程安全的std::queue是在阅读Matrix流量分析源码的时候了解到其自定义的blocking_queue，提供了一种基础的同步机制，通过简单的锁（如互斥锁）来控制多个线程对共享数据的访问。虽然这种实现可以在多线程环境中工作，但它往往不能高效地处理更复杂的同步需求，比如条件阻塞和线程间的通信等。

接下来我们将逐步分析blocking_queue，并对齐进行改造，使之可以应对更加复杂的环境。

## 二、Matrix中的blocking_queue

源码位于：matrix/matrix-android/matrix-traffic/src/main/cpp/util/blocking_queue.h
```cpp
#include <deque>
#include <mutex>

using namespace std;

template <typename T>
class blocking_queue {
private:
    deque<T> coreQueue;
    mutex queueGuard;

public:
    void push(T msg) {
        queueGuard.lock();
        coreQueue.push_back(msg);
        queueGuard.unlock();
    }
    void pop() {
        queueGuard.lock();
        this->coreQueue.pop_front();
        queueGuard.unlock();
    }
    T front() {
        queueGuard.lock();
        T msg = coreQueue.front();
        queueGuard.unlock();
        return msg;
    }
    bool empty() {
        return coreQueue.empty();
    }
    int size() {
        return coreQueue.size();
    }
    void shrink_to_fit() {
        queueGuard.lock();
        coreQueue.shrink_to_fit();
        queueGuard.unlock();
    }
    void clear() {
        queueGuard.lock();
        coreQueue.clear();
        queueGuard.unlock();
    }
};
```
- 问题1：lock&unlock
目前的代码在每个成员函数中都显式地调用 queueGuard.lock() 和 queueGuard.unlock()。这种方法虽然能工作，但不是异常安全的。如果在 lock() 和 unlock() 之间的代码抛出异常，锁可能不会被释放，从而可能导致死锁。建议使用 std::lock_guard 或 std::unique_lock 自动管理锁的获取和释放。

- 问题2：empty&sizeq缺失锁
可能会造成获取的结果返回不正确。

- 问题3：阻塞条件变量的缺失
尽管类名是 blocking_queue，但这个实现中没有使用到条件变量来阻塞等待元素的消费者或生产者，当队列为空时，消费者应该等待直到有元素可用。如果需要实现真正的阻塞队列，应该结合 std::condition_variable 来等待和通知元素的到来或消费。

- 问题4：不合理使用导致问题
来看如看这段代码
```cpp
//thread a
while(!queue.emputy()){
    // 执行某些逻辑
    
    q.front();
}

//thread b
q.pop();
```
在多线程的情况下，可能按序出现如下情况
- thread a先拿到了锁，判断不为空，然后走进了部分自定义的逻辑，此时锁被释放
- thread b此时拿到了锁，并执行了pop方法
- 当a走到front的时候，由于不清楚队列状态发生了变化，则可能出现问题。

## 三、改进
- 改进1：使用RAII机制管理锁。
```cpp
void push(T msg) {
    lock_guard<mutex> guard(queueGuard);
    q_.push_back(msg);
}
```
其他方法类似，C++会根据{}来自动调用unlock，类似于Java中的try-reousrce直接。

- 改进2：补全相关方法的锁
```cpp
bool empty() {
    lock_guard<mutex> guard(queueGuard);
    return q_.empty();
}

int size() {
    lock_guard<mutex> guard(queueGuard);
    return q_.size();
}
```

- 改进3：减少不合理使用导致的问题
```cpp
T try_pop() {
    std::unique_lock<std::mutex> lock(m_);
    if (q_.empty()) {
        return NULL;
    }

    T popped_value = q_.front();
    q_.pop();
    return popped_value;
}
```

- 改进4：增加阻塞条件
结合std::condition_variable来处理
```cpp
std::condition_variable cv_;

void push(const T& data) {
    std::unique_lock<std::mutex> lock(m_);
    q_.push(data);
    lock.unlock();
    cv_.notify_one();
}

T wait_and_pop() {
    std::unique_lock<std::mutex> lock(m_);
    while (q_.empty()) {
        cv_.wait(lock);
    }

    T popped_value = q_.front();
    q_.pop();
    return popped_value;
}

```
此时消费者当发现对位为空，则自动进入等待，push后会进行唤醒。

完整代码:
```cpp
#include <mutex>
#include <queue>

template <typename T>
class concurrent_queue {
private:
    std::queue<T> q_;
    mutable std::mutex m_;
    std::condition_variable cv_;

public:
    void push(const T& data) {
        std::unique_lock<std::mutex> lock(m_);
        q_.push(data);
        lock.unlock();
        cv_.notify_one();
    }

    bool empty() const {
        std::unique_lock<std::mutex> lock(m_);
        return q_.empty();
    }

    T try_front() {
        std::unique_lock<std::mutex> lock(m_);
        if (q_.empty()) {
            return NULL;
        }
        return q_.front();
    }

    T try_pop() {
        std::unique_lock<std::mutex> lock(m_);
        if (q_.empty()) {
            return NULL;
        }

        T popped_value = q_.front();
        q_.pop();
        return popped_value;
    }

    T wait_and_pop() {
        std::unique_lock<std::mutex> lock(m_);
        while (q_.empty()) {
            cv_.wait(lock);
        }

        T popped_value = q_.front();
        q_.pop();
        return popped_value;
    }
};

```
## 四、使用示例

```cpp
constexpr int NUM_PRODUCERS = 3;  // 生产者线程数量
constexpr int NUM_CONSUMERS = 2;  // 消费者线程数量
constexpr int NUM_ITEMS = 10;     // 每个生产者生产的数据数量

// 生产者函数
void producer_func(concurrent_queue<int>& q, int id) {
    for (int i = 0; i < NUM_ITEMS; ++i) {
        int data = id * NUM_ITEMS + i;  // 生成数据
        q.push(data);                   // 将数据放入队列
        std::cout << "生成数据 " << id << " produced: " << data << "\n" << std::endl;
    }
}

// 消费者函数
void consumer_func(concurrent_queue<int>& q, int id) {
    for (int i = 0; i < NUM_ITEMS * NUM_PRODUCERS / NUM_CONSUMERS; ++i) {
        int data;
        data = q.wait_and_pop();  // 从队列中取出数据
        std::cout << "从队列中取出数据 " << id << " consumed: " << data << "\n" << std::endl;
    }
}

int main() {
    concurrent_queue<int> q;  // 创建并发队列

    // 创建生产者线程
    std::vector<std::thread> producers;
    for (int i = 0; i < NUM_PRODUCERS; ++i) {
        producers.emplace_back(producer_func, std::ref(q), i);
    }

    // 创建消费者线程
    std::vector<std::thread> consumers;
    for (int i = 0; i < NUM_CONSUMERS; ++i) {
        consumers.emplace_back(consumer_func, std::ref(q), i);
    }

    // 等待所有生产者线程结束
    for (auto& producer : producers) {
        producer.join();
    }

    // 等待所有消费者线程结束
    for (auto& consumer : consumers) {
        consumer.join();
    }

    return 0;
}
```