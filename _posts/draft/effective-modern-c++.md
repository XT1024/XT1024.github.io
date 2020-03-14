## 条款一： 理解模板类型推导
```c++
template<typename T>
void f(ParamType param);

// 调用时
f(expr);
```
### 场景一：ParamType是指针或引用，非万能引用
1. 忽略`expr`引用部分
2. 对`expr`，`ParamType`执行模式匹配，决定`T`的类型

```c++
template<typename T>
void f(T& param);

// 定义变量
int x = 10;
const int cx = x;
const int& rx = x;

// 例子
f(x);	// T类型为int, param类型为int&
f(cx);	// T类型为const int, param类型为const int&
f(rx);	// T类型为const int, param类型为const int&
```
保持`T`的常量性（如果实参带const）

### 场景二：ParamType是万能引用
1. expr为左值，T和param类型均为`左值引用类型`，**模板类型推导中被推导为引用类型的唯一情形**
2. expr为右值，则按上一情形（忽略引用，执行模式匹配）
```c++
template<typename T>
void f(T&& param);

// 定义变量
int x = 10;
const int cx = x;
const int& rx = x;

// 例子
f(x);	// T类型为int&, param类型为int&
f(cx);	// T类型为const int&, param类型为const int&
f(rx);	// T类型为const int&, param类型为const int&
f(27);	// T类型为int, param类型为int&&
```

### 场景三： ParamType非指针也非引用
非指针也非引用，即`T param`按值传递，param是实参的一个副本。
1. 忽略实参expr的引用类型
2. 若实参expr类型存在`const`, `volatile`,也忽略这些属性
```c++
template<typename T>
void f(T param);

// 定义变量
int x = 10;
const int cx = x;
const int& rx = x;

// 例子
f(x);	// T类型为int, param类型为int
f(cx);	// T类型为int, param类型为int
f(rx);	// T类型为int, param类型为int
```
**由于是按值传递，实参`expr`的`const`,`volatile`不应该影响副本的属性，因此这些属性需要忽略**

### 重点总结
1. 万能引用形参类型推导时，实参为`左值`， `T`将被推导为`左值引用类型`
2. 模板类型推导，具有引用的实参将被当成非引用类型处理（万能引用特殊考虑）
3. 按值传递时，`const`, `volatile`等属性会被忽略。

## 条款二：理解auto类型推导
auto类型推导与模板类型推导规则基本一致，除了初始化列表这种特殊情况，先介绍一般场景
```c++
auto x = 10;	// x为int类型
const auto cx = x; // cx为const int类型
const auto& rx = x; // rx为const int& 类型
auto&& ux = x;	// ux为const int& 类型
auto&& ux = 10; // ux为int&&类型

// => 效果类似于,根据条款一推导

template<typename T>
void special_func(T x); 
special_func(10); // T为int类型，x为int类型

template<typename T>
void special_func_c(const T cx);
special_func_c(x); // T为int类型, cx为const int类型

template<typename T>
void special_func_r(const T& cx);
special_func_r(x); // T为int类型，rx为const int&类型

template<typename T>
void special_func_u(T&& ux);
special_func_u(x); // T为int&类型，ux为int&类型
special_func_u(10); // T为int类型，ux为int&&类型
```
### 初始化列表
`auto`与`模板类型`推导唯一的区别在于，auto将假定大括号的初始化表达式默认代表一个`std::initializer_list`类型，而模板推导则不会。
```c++
auto a = {9, 10, 11}; // a为std::initializer_list类型

template<typename T>
void f(T param);	 
f({9, 10, 11});			// 错误，无法推导T的类型。
```

### 特殊情况
```c++
// c++14

auto create() {
	return {9, 10, 11}; // 错误：模板类型推导失败
}
auto reset = [](const auto& val) {.....}
reset({9, 10, 11}); // 错误： 模板类型推导失败
```
c++14中， 函数返回值或者lambda表达式的形参中使用auto，表明使用模板类型推导，而不是auto推导规则。
### 重点总结
1. `auto`与`模板类型`推导规则基本一致。
2. 不同点在于`auto`将初始化表达式默认代表`std::initializer_list`类型。
3. 模板函数返回值、lambda形参中使用`auto`，表示使用`模板类型`推导，而不是`auto`。（c++14）
