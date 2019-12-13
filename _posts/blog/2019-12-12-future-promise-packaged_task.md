---
layout: post
title: c++ future, promise, packaged_task
category: blog
description: future, promise, packaged_task使用
---

## c++11 future, promise, packaged_task

### std::future
`std::future`是c++11中获取异步操作的结果对象，std::future一般都是由其他异步操作返回的（std::async, std::promise, std::packaged_task）举例说明
```c++
#include<future>
#include<thread>
#include<iostream>
#include <utility>

int main(int argc, char** argvs) {
// 定义一个函数的封装
std::packaged_task<int()> task([]{
	sleep(3);
	return 5;
});
// 获取task的future，当上述函数执行完成以后，future可以获取函数返回的结果。
std::future<int> packaged_future = task.get_future();
// 在线程中执行上述task
std::thread(std::move(task));

int result = packaged_future.get(); // get方法阻塞等待task函数的结束，并且获取其返回值。
std::cout << "task result is " << result << std::endl;
return 0;
}


std::future<int> future = task.get_future();
```
由上述例子可以看出，`std::future<int>` 被生成以后，可以阻塞等待函数结束并且获取其返回值。除此以外std::future还提供其他的函数。
（根据cppreference描述，std::future通过一个shared state与`函数封装`(std::async, std::promise, std::packaged_task)关联，从而实现结果的获取。
> `get` 阻塞获取函数的返回值
> `valid` 判断是否关联shared state
> `wait` 阻塞等待结果执行完成
> `wait_for` 阻塞等待结果执行完成（指定等待时间间隔）,返回当前状态
> `wait_until` 阻塞等待结果执行完成（指定等待至时间点），返回当前状态

总结： std::future提供了获取异步操作执行`结果`的机制，并且可以阻塞获取结果状态。 拿到绑定后的std::future，等于拿到了一张喜茶的叫号单，随时可以查询你的订单当前的状态和结果，无论是看完电影再来取(执行完后调用get)，还是选择在店里等饮料做好（wait, get）都可以随你喜欢。


### std::promise
std::promise<typename T> 是c++11中异步操作结果的封装，std::promise<typename T>在thread1中保存一个类型为T的结果，在给T设值的时候将可以唤醒与其绑定的std::future，并且std::future可以获取其结果。

```c++
#include <thread>
#include <future>
#include <utility>
void Add(int x, int y, std::promise<int> promise) {
	sleep(2);
	promise.set_value(x+y); // 唤醒等待中的std::future
}

int main(int argc, char** argvs) {
	std::promise<int> promise;
	std::future<int> promise_future = promise.get_future();
	// promise 传入线程中，并且开始执行Add函数
	std::thread(Add, 5, 6, std::move(promise));
	int result = promise_future.get() // 在promise set_value前一直阻塞等待结果。
	std::cout << "result " << result << std::endl;
	return 0;
} 
```
> std::promise 承诺给你一个 std::future，在那之前，请你务必等我（阻塞)。 是c++程序员的浪漫没错了。

### std::packaged_task
TODO....