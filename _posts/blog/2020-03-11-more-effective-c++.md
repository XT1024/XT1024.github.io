---
layout: post
title: more-effective-c++ 读书笔记
category: blog
description: 记录more-effective-c++的条款理解
---

## 条款15：只要有可能使用constexpr，就使用它

> `constexpr`可用于声明对象和函数，声明对象时其值必须是编译期可知的，而声明函数则如果传入的参数是`constexpr`，函数结果也会在编译期计算出

### 声明对象
```c++

constexpr auto nums = 5; // ok
std::array<int, nums> arr; // ok nums为编译期结果，

int w;
constexpr auto fool = w; // wrong, w编译期不可知
std::array<int, w> bad; // wrong, w为运行期变量，并非常量表达式

```
`constexpr` 对象均是`const`，并且必须是编译期可知的内容。

### 声明函数

c++11中`constexpr`的内容不能多于一条`return`语句，c++14中放开了这个限制。
```c++
constexpr int add(int i, int j) noexcept {
	return i+j;
}

constexpr auto result = add(5, 6); // ok, add结果编译期计算

// constexpr用于构造函数以及类成员函数
class Circle {
public:
	constexpr Circle(double r): m_radius(r) {}

	constexpr double GetRadius() {
		return m_radius;
	}
	/*	由于c++11中constexpr要求返回不能是void，并且其修饰的函数默认有const属性，因此SetRadius不能是constexpr,c++14中没有这两个限制。
		constexpr void SetRadius(double r) {
			m_radius = r;
		}
	*/
private:
	double m_radius;
};
```
`constexpr`声明的函数可以是构造函数,或者是类成员函数

1. 内容不能多于一条语句(c++11)
2. 返回值和参数都需要是`literal type` 非void (c++11)
3. 修饰的函数自带`const`属性(c++11)



## 条款16：保证const成员函数的线程安全
`const`成员函数 + `mutable`成员时，需要考虑线程安全性（用std::atomic或者std::mutex保护）