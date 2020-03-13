---
layout: post
title: more-effective-c++ 读书笔记
category: blog
description: 记录more-effective-c++的条款理解
---

## 条款13： 优先选用const_iterator
优先选择`const_iterator`，如下
```c++
std::vector<int> values;
auto it = std::find(values.cbegin(), values.cend(), 1968);
values.insert(it, 1998);
```
 
 然而在模板中，一般会使用非成员函数版本的`cbegin`与`cend`, c++11目前只提供了`std::begin`,`std::end`(短视hhh)，所以需要自己想办法实现`std::cend`
 Meyers给出了一个巧妙的实现
```c++
template<typename T>
auto cbegin(const T& container)->std::decltype(std::begin(container)) {
	return std::begin(container);
}
template<typename T>
auto cend(const T& container)->std::decltype(std::end(container)) {
	return std::end(container);
}
```
上述实现实际上是利用了`std::begin`和`std::end`对于传入的是参数是`const`类型，将会返回其`const_iterator`, 因此可以利用形参为const来转换。


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



## 条款17：默认特种成员函数的生成机制
`默认构造函数`，`默认析构函数`，`默认复制构造函数`,`默认复制赋值函数` 在用户未自行定义的情况下编译器将会自动生成，这点应该都很清楚。
c++11中新增的`移动构造函数`，`移动赋值函数`的生成机制有些差异，具体的有点复杂，记住没有什么意义。

>在编写代码的时候，应当显式的定义所需要的特种成员函数

```c++
class Base {
public:
	Base() = default;
	Base(const Base& base) = default;
	Base& operator=(const Base& base) = default;
	
	Base(Base&& base) = default;
	Base& operator=(Base&& base) = default;

};
```

## 条款18：使用std::unique_ptr管理具备专属所有权资源
`std::unique_ptr`优先选用，可认为大小与裸指针相同，支持传用户自定义deleter，如下

```c++
auto deletor = [](Object* obj) {
	doSomething(obj);
	delete obj;
};
std::unique_ptr<Object, std::decltype(deletor)> p(nullptr, deletor);
```
`std::unique_ptr`提供两种形式:
1. `std::unique_ptr<T>` 针对单个对象，不提供`[]`操作符
2. `std::unique_ptr<T[]>` 针对数组，

**`std::unique_ptr` 可以方便的转换为 `std::shared_ptr`**

## 知识性了解(TODO)
1. 为何要提供两种形式
2. 工厂函数的实现可以自己尝试下

## 条款19：使用std::shared_ptr管理具备共享所有权的资源
不能过于滥用`std::shared_ptr`！！！！（考虑km提到的在lambda里的使用问题
>思考：使用单例shared_ptr是否是最佳选择？