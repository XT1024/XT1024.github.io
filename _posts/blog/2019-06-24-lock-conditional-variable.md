---
layout: post
title: 互斥锁与条件变量
category: blog
description: 记录互斥锁与条件变量的使用方式，以及常见的工程应用
---

## 互斥锁与条件变量

### 互斥锁
互斥锁是用于线程间同步的控制量，避免发生多线程间读写的冲突。

```c++
						// |              thread_1                           	||        thread_2				||
int a = 1;	// | _ _ _ _ || _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ ||_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ ||
a = a + 1;	// |int tmp = a + 1; cpu时间被切到thread_2        		||int tmp1 = a + 1;切回thread_1 	||	
						// |a = tmp; 此时a = 2; thread_1执行完成，切回到thread_2	|| a = tmp1; 				   	|| 
						// -----------------------------------------------------||------------------------------
						//  thread_1 thread_2执行完后a = 2 (而本意是thread_1 和 thread_2各执行一次)
```
由于线程可能执行到任意阶段，被操作系统切换到其他执行线程上，由此会导致多个线程共享状态时出现冲突。

 > `互斥锁`则是用于线程间同步的手段，只有获得互斥锁所有权的线程可继续执行，其余线程将阻塞，直至互斥锁被释放。

**注意：已经获取mutex的线程企图重复获取mutex的行为在c++11的标准中时未定义的，除非mutex本身是可重入类型。**

### 互斥锁的使用
c++11 提供了std::mutex,std::timed_mutex...等互斥量，mutex可移动不可复制，以常用的std::mutex举例，使用std::mutex除了其自身提供的三种函数
> lock: 线程获取mutex，如果mutex已经被其他线程获取，则该线程将阻塞直至其获取到被释放的mutex, 已持有mutex的线程重复调用lock的行为是未定义的。
> try_lock: lock是阻塞式，获取锁失败的线程将无法执行其他任务，而try_lock获取mutex会返回false
> unlock: 获取mutex的线程执行完任务，调用unlock释放mutex.

实际上很少直接使用mutex自身的lock,unlock,而是使用std::lock_guard和std::unique_lock
#### std::lock_guard
std::lock_guard本质上是互斥量RAII的封装，既`构造`时获取锁（lock)`析构`时释放锁(unlock)，**因此std::lock_guard不可复制，不可移动。**
```c++
#include<mutex>
#include<iostream>
class SafeIncrement {
public:
	SafeIncrement(val):m_val(val) {
	}
	void Increment() {
		std::lock_guard<std::mutex> lock(m_mutex); // std::lock_guard构造时获取互斥锁，析构时自动释放
		m_val++;
		// 离开作用域，lock自动释放m_mutex
	}
private:
	std::mutex m_mutex;
	int m_val;
};
```
#### std::unique_lock
为了更灵活的使用互斥锁，c++11进一步提供了std::unique_lock(不可复制，可移动），介绍以下几种常用方式。
```c++
explicit unique_lock( mutex_type& m ); (3)	(since C++11)
// 与std::lock_guard()类似，构造时直接尝试获取锁，析构时释放锁

unique_lock( mutex_type& m, std::defer_lock_t t ) noexcept;(4)	(since C++11)
// 构造unique_lock对象时不获取锁，而是通过std::unique_lock::lock 函数调用触发（析构时尝试释放锁）

unique_lock( mutex_type& m, std::try_to_lock_t t ); (5)	(since C++11)
// 构造时尝试使用try_lock方式获取锁,std::unique_lock::owns_lock判断是否获得锁。析构时释放锁（如果获得了的话）

unique_lock( mutex_type& m, std::adopt_lock_t t ); (6)	(since C++11)
// 构造时不尝试获取锁，而是假定当前线程已经获取了锁，（如果线程为获取到锁的行为是未定义的）
```
举个使用unique_lock的例子
```c++
std::mutex mutex_a, mutex_b, mutex_c;
int a,b,c;

void Update() {
	{
		// 效果与std::lock_guard<std::mutex> guard(mutex_a);一致
		// 构造获取mutex_a，离开作用域释放mutex_a
		std::unique_lock<std::mutex> auto_lock(mutex_a);
		a++;
	}
	// 注意：由于使用std::defer_lock，构造时并不尝试获取互斥量
	std::unique_lock<std::mutex> hand_lockb(mutex_b, std::defer_lock);
	std::unique_lock<std::mutex> hand_lockc(mutex_c, std::defer_lock);
	// 正式获取两个互斥量（std::lock是保证获取两个互斥量时不会出现死锁现象）
	std::lock(hand_lockb, hand_lockc);
	b++;
	c++;
	// 离开了hand_lockb,hand_lockc的作用域，析构时自动释放mutex_b,mutex_c
}
```

### 条件变量
mutex只是保护多线程共享的数据，而在实际的工程应用中，通常会遇到如下的情况
> 消费线程A，B,  C想拿走std::vector中的数据处理，但是std::vector是空的时候是无法处理的，这时候需要等待生产线程D往std::vector中写入数据，并且通知A,B,C可以消费。

上述场景仅靠mutex是无法实现的，因此c++11提供了std::condition_variable
#### std::condition_variable
std::condition_variable是实现线程间同步的控制量，用于阻塞一个或多个线程，直到另一个线程修改条件变量并且通知condition_variable。并且condition_variable必须搭配mutex使用。

condition_variable 主要有两种行为，wait, notify。
函数名 | 描述 |  是否需要获取mutex | 备注  
-|-|-|-
wait | 线程沉睡，并且进入条件变量的等待队列等待唤醒 | 必须使用std::unique_lock | 执行完wait函数内自动释放mutex|
wait_for| 线程沉睡，并且进入条件变量的等待队列等待唤醒，如果超过设定时间未被成功唤醒，则自动醒来并返回false | 必须使用std::unique_lock | 进入沉睡后自动释放mutex|
wait_until | 与wait_for功能类似，不过wait_util是传入结束等待的时间点 | 必须使用std::unique_lock |进入沉睡后自动释放mutex|
notify_one | 唤醒一个等待在该条件变量上的线程 | 不需要获取互斥量 | **最好在线程互斥量释放后调用，避免虚假唤醒** |
notify_all | 唤醒所有等待在该条件变量上的线程，未获得互斥量的线程重新沉睡，回到等待队列 | 不需要获取互斥量| **最好在线程互斥量释放后调用，避免虚假唤醒**| 


### Example
```c++
#include <condition_variable>
#include <mutex>
#include <vector>

template <typename T>
class SharedQueue {
public:
    SharedQueue(int length) : m_max_length(length) {}
    ~SharedQueue() {}
    void Push(const T& item) {
        // wait 需要搭配unique_lock使用
        std::unique_lock<std::mutex> lock(m_mutex);
        m_not_full.wait(lock, [this] { return !IsFull(); });
        m_data.emplace_back(item);
        lock.unlock();  // 提前手动释放，避免notify_one虚假唤醒
        m_not_empty.notify_one();
    }
    T Pop() {
        // unique_lock 同样RAII机制，析构自动释放互斥量
        std::unique_lock<std::mutex> lock(m_mutex);
        m_not_empty.wait(lock, [this] { return !IsEmpty(); });
        T t = m_data.front();
        m_data.pop();
        lock.unlock();  // 提前手动释放，避免notify_one虚假唤醒
        m_not_full.notify_one();
        return t;
    }

    bool Empty() {
        std::lock_guard<std::mutex> lock(m_mutex);
        return m_data.empty();
    }

    bool Full() {
        std::lock_guard<std::mutex> lock(m_mutex);
        return m_data.size() == m_max_length;
    }

private:
    bool IsFull() { return m_data.size() == m_max_length; }

    bool IsEmpty() { return m_data.empty(); }

private:
    std::vector<T> m_data;
    int m_max_length;
    std::condition_variable m_not_empty;
    std::condition_variable m_not_full;
    std::mutex m_mutex;
};
```