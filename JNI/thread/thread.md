## 一、基本使用
在 C++11 中，线程支持是通过 <thread> 头文件中的 std::thread 类提供的。
下面是一个简单的例子，展示如何使用 std::thread 来创建和管理线程。
```cpp
#include <iostream>
#include <thread>
#include <chrono>

// 打印字母的函数
void print_letters() {
    for (char c = 'A'; c <= 'Z'; ++c) {
        std::cout << c << " ";
        std::this_thread::sleep_for(std::chrono::milliseconds(100)); // 稍微延时以便于观察输出
    }
    std::cout << std::endl;
}

// 打印数字的函数
void print_numbers() {
    for (int i = 1; i <= 26; ++i) {
        std::cout << i << " ";
        std::this_thread::sleep_for(std::chrono::milliseconds(100)); // 稍微延时以便于观察输出
    }
    std::cout << std::endl;
}

int main() {
    // 创建两个线程
    std::thread thread1(print_letters);
    std::thread thread2(print_numbers);

    // 等待这两个线程完成它们的任务
    thread1.join();
    thread2.join();

    return 0;
}

```

- 线程创建：使用 std::thread 类的构造函数创建了两个线程。构造函数的参数是一个可调用对象，这里分别是 print_letters 和 print_numbers 函数。
- join()：join() 方法用于等待线程完成其执行。调用 join() 的主线程（在这个例子中是 main() 函数）将阻塞，直到对应的线程结束其执行。这是确保程序在所有线程完成工作之前不会退出的常用方法。
## 二、源码

- 头文件：libstdc++-v3/include/bits/std_thread.h
- 实现：libstdc++-v3/src/c++11/thread.cc

## 三、STL实现
### 3.1 构造方法
下面代码是std::thread的带参构造行数，用于创建一个线程来执行某个函数（或者叫可调用对象）。
```cpp
/**
 * _Callable 是一个可调用类型，可以是函数指针、函数对象或 lambda 表达式
 * _Args... 是可调用对象 __f 所需要的参数类型
 **/
template<typename _Callable, typename... _Args,
	     typename = _Require<__not_same<_Callable>>>
      explicit
thread(_Callable&& __f, _Args&&... __args){

    //静态断言，检查传入的函数和参数是否匹配
	static_assert( __is_invocable<typename decay<_Callable>::type,
				      typename decay<_Args>::type...>::value,
	  "std::thread arguments must be invocable after conversion to rvalues"
	  );

    //使用 _Wrapper = _Call_wrapper<_Callable, _Args...> 类型来封装可调用对象和它的参数
	using _Wrapper = _Call_wrapper<_Callable, _Args...>;

    //1、这行代码创建了 _State_impl<_Wrapper> 的一个新实例，
    //  使用 std::forward 来完美转发 _Callable 和 _Args... 到 _Wrapper 构造函数中，这样可以避免不必要的复制
    //2、_M_start_thread 是一个负责实际创建和启动线程的内部函数，接受一个状态对象（在这里是 _State_ptr 智能指针，指向 _State_impl<_Wrapper>）和一些关于线程启动行为的额外参数
	_M_start_thread(_State_ptr(new _State_impl<_Wrapper>(
	      std::forward<_Callable>(__f), std::forward<_Args>(__args)...)),
	    _M_thread_deps_never_run);
}


/**
 * 该方法负责启动一个新的线程
 * state:_State_impl<_Wrapper>的智能指针
 * depend：忽略即可，用于避免一些编译问题
 **/
void thread::_M_start_thread(_State_ptr state, void (*depend)())
  {
    // Make sure it's not optimized out, not even with LTO.
    asm ("" : : "rm" (depend));

    //检查环境是否支持多线程，不支持的话抛异常（也是刚了解到,个别编译环境可能没有启动线程支持 - -#）
    if (!__gthread_active_p())
      {
#if __cpp_exceptions
	throw system_error(make_error_code(errc::operation_not_permitted),
			   "Enable multithreading to use std::thread");
#else
	__builtin_abort();
#endif
      }

    // 核心：使用__gthread_create来创建线程，其真实调用的是pthread_create 
    // _M_id._M_thread 线程id,标识，内部是一个pthread_t
    // execute_native_thread_routine: 指向被线程执行的函数指针， 这个指针是用于传递给pthread_create执行，pthread_create在执行的时候会将我们传入待执行的函数指针&参数传递给它。
    // state.get(): 传入到线程中待执行的函数&参数封装体，其最终会在执行的时候传递给execute_native_thread_routine，由其负责执行
    const int err = __gthread_create(&_M_id._M_thread,
				     &execute_native_thread_routine,
				     state.get());
    //线程创建失败，抛出系统错误
    if (err)
      __throw_system_error(err);
    state.release();
}

```

### 3.2 join

thread::join方法用于阻塞调用线程，直到传给线程的方法执行完成。

```cpp

void  thread::join() {
    // 预设一个参数异常的错误
    int __e = EINVAL;

    // _M_id是线程标识，id()获取到的是默认构造id,标识无效或者是空线程。如果两者不相等，说明此线程是一个活跃可进行join的线程。
    if (_M_id != id())
      //将线程的标识传入，底层调用join相关的方法
      __e = __gthread_join(_M_id._M_thread, 0);

    //错误处理
    if (__e)
      __throw_system_error(__e);

    //线程被join后，其id会被重置，按照上面的逻辑，同时也意味着无法两次join
    _M_id = id();
}
```
注意，每个 std::thread 对象只能被 join() 一次。重复 join() 会抛出异常。

### 3.3 detach
detach()是与join()相对的方法。线程在执行detach()后独立运行，脱离创建者，其生命周期对象不受原始的thread对象控制。
```cpp
//整体流程和join方法类似，这里就不多做介绍了
void  thread::detach() {
    int __e = EINVAL;

    if (_M_id != id())
      __e = __gthread_detach(_M_id._M_thread);

    if (__e)
      __throw_system_error(__e);

    _M_id = id();
}
```
注意，分离线程时需要确保线程的执行过程中不会访问任何已经销毁或作用域结束的对象。
### 3.4 析构方法
从代码上看，其析构方法非常简单，设计核心就是确保线程在析构的时候已经被join或detach过，否则会在thread销毁的时候抛出异常。一定程度上是为了避免不必要的资源泄漏或错误。
```cpp
~thread(){
    if (joinable())
	std::__terminate();
}

//当thread调用join()或detach)时，_M_id == id()
bool joinable() const noexcept { 
    return !(_M_id == id()); 
}
```
### 3.5 拷贝与移动

#### 3.5.1 拷贝
```cpp
thread(const thread&) = delete;
```
拷贝构造函数在std::thread中是被删除的，这样的操作确保了可操作线程的唯一性，减少了资源管理的复杂程度，可能造成一处创建，多处停止等行为。


#### 3.5.2 移动
前面提到的copy，是多个对象操作同一个thread，但是移动不同，是直接将控制权移交，保证了可操作线程的唯一性。
```cpp

//移动构造函数，交换一下线程标识id即可
thread(thread&& __t) noexcept
{ swap(__t); }

//移动赋值函数，如果当前线程已经join或detach，则不允许
thread& operator=(thread&& __t) noexcept {
      if (joinable())
	std::__terminate();
      swap(__t);
      return *this;
}

void swap(thread& __t) noexcept
{ std::swap(_M_id, __t._M_id); }
```

为何移动赋值函数需要判断joinable，而移动构造不需要 ？ 
- 移动构造：它是在创建新对象时被调用的，此时接收资源的对象是新实例化的，尚未持有任何资源。因此，不需要检查目标对象是否已经拥有一个线程（即它是否 joinable），因为新创建的对象默认不会拥有任何线程资源。
- 移动赋值：它是在已存在的对象上操作的，这意味着目标对象可能已经拥有一个实际对应且操作过线程。如果直接覆盖，可能造成对之前的底层的线程失去操作权，造成泄漏。
故动赋值函数需要判断joinable，而移动构造不需要。