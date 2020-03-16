---
layout: post
title: more-effective-c++ 读书笔记
category: blog
description: 记录more-effective-c++的条款理解
---
## 条款10：优先选用scoped的枚举类型，而非不限作用域的枚举类型
```c++
enum Color {black, white, red};
auto white = false; // wrong

enum class Color {black, white, red};
auto white = false; // ok
Color d = white;	// wrong
Color c = Color::white; // ok

```
简单理解：`enum class`是带作用域的强类型，不会隐式的转换到整数类型。


## 条款11: 使用删除函数，而不是用private
c++11中新增删除函数，即`= delete`用于声明函数未定义，主要在以下场景中使用
1. 声明特种函数为`未定义`状态，例如不需要复制构造函数可声明为`= delete`
2. 在一般函数，或者模板函数中，声明某种参数类型的函数为`= delete`

## 条款12： 为意在改写的函数添加override声明
`override`声明表示虚函数的改写，可以避免因为错误的声明了函数签名。


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

## 条款21： 优先使用std::make_shared和std::make_unique，而不是使用new
**理由一：减少重复输入类名**
```c++
auto ptr(std::make_shared<Widget>());
std::shared_ptr<Widget> p(new Widget);
```
**理由二：潜在的内存泄漏**
```c++
void ProcessWidget(std::shared_ptr<Widget>(new Widget), compute());
/*
编译器产生的代码顺序可能是
1. new Widget
2. compute()
3. std::shared_ptr<Widget>(ptr)
*/
void ProcessWidgetOk(std::make_shared<Widget>(), compute()); // ok
```
由于编译器可能产生上述代码，如果在`compute`中产生了异常，则`3`不会执行，导致内存泄漏.

**理由三： 性能提升**
`std::shared_ptr`实际上会产生两次内存分配，一次是所指对象的内存，另一次是引用技术的控制块内存，而`std::make_shared`会一次申请两个的内存。

### 不适合使用make系列的场景
**场景一： make系列不能使用自定义析构器**
**场景二：大括号初始化物的构造函数匹配问题**
**场景三： shared_ptr: 自定义内存管理的类；内存紧张、大对象、存在生存周期长于shared_ptr的weak_ptr**

## 条款23：理解std::move和std::forward
`std::move`并不移动什么，本质上只是一个强制右值转换的函数，`std::forward`是有条件的强制右值转换。

### std::move
**理解一：std::move仅是一个强制右值转换的函数**
低配版本的move实现
```c++
template<typename T>
typename std::remove_reference<T>::type&& // make sure return rvalue
move(T&& param) {
	using ReturnType = std::remove_reference<T>::type&&;
	return static_cast<ReturnType>(param);
}
```
**理解二：std::move不应该接受一个const的参数**
`std::move`在强制转换为右值时保留`const`，因此其函数结果返回的是一个`const`，函数返回结果作为参数，并不会调用想象中的`移动构造函数`，而是调用了`复制构造函数`
```c++
class Widget {
public:
	explicit Widget(const std::string val):
	m_value(std::move(val)) {}
private:
	std::string m_value;
}
```
上述例子是无效的。。。`m_value(std::move(val))`调用的并不是`移动构造函数`，而是`复制构造函数`。

### std::forward
`std::forward`是一个有条件的强制右值转换，先看下`std::forward`的来源。
```c++
void process(const Widget& widget);
void process(Widget&& widget);

template<typename T>
void log_and_process(T&& t) {
	log("log and process");
	process(t);
}
Widget w;
log_and_process(w);
log_and_process(std::move(w));
```
上述代码本意是在`log_and_process`传入`右值`时调用`process`的右值参数版本，在为`左值`时,调用`process`的左值参数版本。可惜的是上述调用的都是`process(const Widget& widget)`，因为形参`t`必定是`左值`！！！！

```c++
void process(const Widget& widget);
void process(Widget&& widget);

template<typename T>
void log_and_process(T&& t) {
	log("log and process");
	process(std::forward<T>(t)); // ok
}
Widget w;
log_and_process(w);
log_and_process(std::move(w));
```
上述`std::forward`作用是当`log_and_process`传入的是`右值`情况下，将`形参t`强制转换成`右值`。
```c++
template<typename T>
void f(T&& t) {
	process(std::forward<T>(t))
}
int i = 1;
f(i);	// T为 int&, t为 int&
f(27);	// T为 int, t为 int&

template<typename T>
T&& forward(typename std::remove_reference<T>::type& t) { 
// f传入左值返回左值类型，传入右值返回右值类型
	return static_cast<T&&>(t);
}
```
推导过程参看 [forward](http://xt1024.github.io/forward)

## 条款24：区分万能引用和右值引用（！！！！尤其注意！！！）
`universal reference`必须正好是`T&&`，其他的均为右值引用。


## 条款25：针对右值引用实施std::move，针对万能引用实施std::forward
**理解一: 确定是右值引用时，使用`std::move`**

```c++
class Widget {
public:
	void setName(std::string&& name) {
		m_name = std::move(name);
	}
}
```
**理解二: 万能引用时，使用`std::forward`转发（可能是传入左值引用，如果用`std::move`的话破坏传入的值而外部感知不到。）
```c++
class Widget {
public:
	template<typename T>
	void setAttribute(T&& param) {
		m_name = std::forward<T>(param);
	}
}
```
**理解三： 千万别干在函数返回值中使用`std::move`的神奇操作，弄巧成拙**
```c++
Widget make(const std::string& name) {
		Widget w;
		w.setName(name);
		return std::move(w); // 绝对不要干这种事，编译器有return value optimization(RVO) 极有可能破坏了编译器优化，参考`RVO` 第二条
}

```
`RVO`标准有两个前提条件
1. 局部对象类型和函数返回值类型相同
2. 返回的就是局部对象本身
实际上，标准要求：当`RVO`的前提条件允许，要么发生`复制省略`，要么`std::move`隐式的被实施于返回的局部对象上。


## 条款26： 避免依万能引用类型进行重载
1. 万能引用会劫持大部分参数，在大部分情况下将发生意想不到的调用版本。
2. 尽量别搞完美转发构造函数，真的，除非你很清楚会发生什么。


## 条款27： 万能引用进行重载的替代方案（这个条款比较有趣，可以读读，但尽量别用）


## 条款28： 理解引用折叠
`引用折叠`是编译器自身转换的原则，不代表开发者可以使用。 理解引用折叠是理解`std::forward`,`std::move`源码的基础。
**理解一： 引用折叠是编译器视角发生的事**
**理解二： 原始的引用中存在任意一个是左值引用，则结果为左值引用；否则为右值引用**