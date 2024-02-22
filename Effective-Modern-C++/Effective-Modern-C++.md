# Item1:模板类型推断

```cpp
template<typename T>
void f(ParamType param);
f(expr);
```

在编译期，编译器使用`expr`推断两类型:

- `T`
- `ParamType`:往往包含`const`、`&`

```cpp
template<typename T>
void f(const T& param); // ParamType is const T&
int x = 0;
f(x);
```

`f(x)`：`T`推断为`int`，`ParamType`推断为`const int&`

## 情况1：`ParamType`是引用或指针类型，但不是广义引用(&&)

- `expr`是引用时，**忽略其`&`**，然后与`ParamType`模式匹配来确定`T`,如：

```cpp
template<typename T>
void f(T& param);

int x = 27;
const int cx = x;
const int& rx = x;

f(x); // T is int, param's type is int&
f(cx);  // T is const int, param's type is const int&
f(rx);  // T is const int, param's type is const int&
```

推断期间，下面`rx`的`&`同样被忽略

```cpp
template<typename T>
void f(const T& param);

int x = 27;
const int cx = x;
const int& rx = x;

f(x); // T is int, param's type is int&
f(cx);  // T is int, param's type is const int&
f(rx);  // T is int, param's type is const int&
```

右值引用同理，但只有右值实参才会被赋给右值引用形参

- `expr`是指针(或pointer to const)时同理

```cpp
template<typename T>
void f(T* param);

int x = 27;
const int *px = &x;

f(&x);  // T is int, param's type is int*
f(px);  // T is const int, param's type is const int*
```

## 情况2：`ParamType`是广义引用

- `expr`是左值时，`T`和`ParamType`都推断为**左值引用**（这是在模板类型推断中，`T`被推断为引用的**唯一情况**）

```cpp
template<typename T>
void f(T&& param);

int x = 27;
const int cx = x;
const int& rx = x;

f(x); // x is lvalue, T is int&, param's type is int&
f(cx);  // cx is lvalue, T is const int&, param's type is const int&
f(rx);  // rx is lvalue, T is const int&, param's type is const int&
f(27);  // 27 is rvalue, T is int, param's type is int&&
```

- `expr`是右值时，使用**情况1**

## 情况3：`ParamType`非指针非引用

```cpp
template<typename T>
void f(T param);
```

`param`始终**按值传递参数**（Copy一个)，忽略`expr`类型中**实参本身**的`& const volatile`修饰词

```cpp
int x = 27;
const int cx = x;
const int& rx = x;

f(x); // T is int, param's type is int
f(cx);  // T is int, param's type is int
f(rx);  // T is in&, param's type is int
```

`const volatile`仅仅在**实参本身按值传递时**才被忽略（实参本身的修饰），当实参为指针或引用时，其所指向的对象的这些修饰不能忽略，如：

```cpp
template<typename T>
void f(T param);

const char* const ptr = "hello";
f(ptr); // ptr本身按值传递，修饰ptr的const忽略掉，而其指向的是const char*，指向对象的修饰不能被忽略，所以T is const char*
```

### 数组作为实参

数组和指针是**不同的类型**，但数组可退化为指针：

```cpp
const char name[] = "abc";  // name's type is const char [4]
const char * ptrToName = name; // ptrToName's type is const char *
```

当数组作为实参时，按值传递的模板`T`被推断为指针：

```cpp
template<typename T>
void f(T param);

f(name);  // T is const char *
```

按引用传递的模板`T`被推断为数组类型：

```cpp
template<typename T>
void f(T& param)

f(name);  // T is const char [4], param's type is const char (&)[4]

template<typename T, std::size_t N>
constexpr std::size_t arraySize(T (&)[N]) noexcept
{
    return N; // N 在编译期求出 为 4
}
```

### 函数作为实参

函数类型可退化为函数指针,

- 模板按值传递时，退化为指针
- 模板按引用传递时，不退化

```cpp
void someFunc(int, double);    // type is void(int,double)

template<typename T>
void f1(T param);

template<typename T>
void f2(T& param);

f1(someFunc)   // T is  void (*)(int, double)
f2(someFunc).  // T is  void (int, double)
```

## 总结

- 模板类型推断时，若实参是引用类型，则忽略其引用
- 推断按值传递的参数时，`const`和`volatile`被忽略
- 函数和数组在按值传递的模板中退化为指针

# Iterm2:auto 类型推断

`auto`类型推断就是模板类型推断，`auto`充当着模板推断中`T`的作用（区别：auto 将花括号括的数列当作std::initializer_list，但模板类型推断无法判断）

- case1: 类型修饰词是指针或引用，不是广义引用
- case2:类型修饰词时广义引用
- case3:类型修饰词不是指针或引用

```cpp
auto x = 2;              // case 3
// 等价的模板
template<typename T>
void func_for_x(T param);
func_for_x(2);           // param's type is x's type

const auto cx = x;       // case 3
// 等价的模板
template<typename T>
void func_for_cx(const T param);
func_for_cx(x);         // param's type is cx's type

const auto& rx = x;     // case 1
// 等价的模板
template<typename T>
void func_for_rx(const T& param);
func_for_rx(x);        // param's type is rx's type

// 广义引用
auto&& uref1 = x;      // x is lvalue, uref1's type is int&
auto&& uref2 = cx;     // cx is const lvalue, uref2's type is const int&
auto&& uref3 = 27;     // 27 is rvalue, uref3's type is int&&

// 使用 auto，数组和函数退化为指针
const char name[] = "abc";    // name' s type is const char[4];
auto arr1 = name;             // arr1's type is const char*
auto& arr2 = name;            // arr2's type is const char(&)[4]

void someFunc(int, double);
auto func1 = someFunc;        // func1's type is void(*)(int, double)
auto& func2 = someFunc;       // func2's type is void(&)(int, double)

// 唯一的例外
auto a = 27;
auto b(27);   // int

auto c = {27}; // std::initializer_list<int>
auto d{27};    // std::initializer_list<int>
```

当`auto`作为函数返回类型或 lambda 表达式的形参时，等价于**模板类型推断**而不是auto类型推断

# Item3:理解 decltype

取得表达式或名称的类型

```cpp
const int i = 0;         // decltype(i) is const int
bool f(const Widget& w); // decltype(w) is const Widget&
                         // decltype(f) is bool(const Widget&)
struct Point {
 int x, y;               // decltype(Point::x) is int
};                       // decltype(Point::y) is int
```

在函数模板中，返回值的类型依赖于参数的类型时(C++11)：

```cpp
template<typename Container, typename Index> // works, but
auto authAndAccess(Container& c, Index i)    // requires
 -> decltype(c[i])                           // refinement
{
	 authenticateUser();
	 return c[i];
}
```

C++14支持自动推断函数和lambda的返回类型：

```cpp
template<typename Container, typename Index> // C++14;
auto authAndAccess(Container& c, Index i)    // not quite
{                                            // correct
	 authenticateUser();
	 return c[i];                              // return type deduced from c[i]
}
```

但当`auto`应用于返回类型时，采用的是**模板类型推断方法**，此时引用和const会被忽略，对于`vector<int>` 来说，将返回`int` 类型，而我们期望的是`int&` 。

此时，我们需要使用 **`decltype`** 类型推导来指定其返回类型，即指定 **`authAndAccess`** 应该返回与表达式 **`c[i]`** 完全相同的类型。在 C++14 中，C++ 的设计者们为了应对在一些需要推断类型的情况下使用 **`decltype`** 类型推导规则的需求，引入了 **`decltype(auto)`** 说明符。auto 表明类型应该被推导，而 decltype 表示在推导期间应该使用 decltype 的规则：

```cpp
template<typename Container, typename Index> // C++14; works,
decltype(auto)                               // but still
authAndAccess(Container& c, Index i)         // requires
{                                            // refinement
	 authenticateUser();
	 return c[i];
}
```

在声明变量时：

```cpp
Widget w;
const Widget& cw = w;
auto myWidget1 = cw;           // auto type deduction: 忽略const、&, type is Widget
decltype(auto) myWidget2 = cw; // decltype type deduction: type is const Widget&
```

上面`authAndAccess` 的问题是：无法接收右值容器，解决办法是：

- 使用函数重载：左值引用和右值引用两个版本，太麻烦
- 使用广义引用参数

```cpp
// final C++14 version
template<typename Container, typename Index>
decltype(auto)
authAndAccess(Container&& c, Index i)
{
	 authenticateUser();
	 return std::forward<Container>(c)[i];
}

// final C++11 version
template<typename Container, typename Index>
auto // C++11
authAndAccess(Container&& c, Index i)
-> decltype(std::forward<Container>(c)[i])
{
	 authenticateUser();
	 return std::forward<Container>(c)[i];
}
```

特殊情况，将**`decltype`** 用于左值表达式时，其类型是`**&` ：**

```cpp
int x = 0;
// decltype(x) is int
// decltype((x)) is int&, x is lvalue
// decltype((x+1)) is int&&, x+1 is rvalue
```

# 推断类型的方法

```cpp
template<typename T> // 只有声明 TD 的类模板;
class TD; // TD == "Type Displayer"
```

尝试实例化这个模板会引发错误消息，因为没有模板定义可以实例化。为了查看 x 和 y 的类型，只需尝试用它们的类型实例化 TD：

```cpp
TD<decltype(x)> xType; // 引发包含 x 的类型的错误
TD<decltype(y)> yType; // 引发包含 y 的类型的错误
```

运行时打印类型：

```cpp
std::cout << typeid(x).name() << '\\\\\\\\n'; // display types for
std::cout << typeid(y).name() << '\\\\\\\\n'; // x and y

template<typename T>
void f(const T& param)
{
	  using std::cout;
		cout << "T = " << typeid(T).name() << '\\\\\\\\n';         // show T
		cout << "param = " << typeid(param).name() << '\\\\\\\\n'; // show param's type
    …
}
```

# Item5:使用auto声明类型

```cpp
template<typename It>
void dwim(It b, It e) {
    while(b != e) {
	    // 太长
        // typename std::iterator_traits<It>::value_type currValue = *b;
        auto currValue = *b;
        ...
    }
}
// function太麻烦
std::function<bool(const std::unique_ptr<Widget>&, const std::unique_ptr<Widget>&)>
derefUPLess
	= [](const std::unique_ptr<Widget>& p1, const std::unique_ptr<Widget>& p2)
    { return *p1 < *p2; };
```

C++14以后的 lambda：

```cpp
auto derefLess =
[](const auto& p1,const auto& p2) // values pointed
{ return *p1 < *p2; }; // to by anything
```

`auto` 声明的`derefLess` 就是闭包的类型，这与 `std::function`声明的类型不同，`std::function`类模板实例对象有一个固定的大小，当大小不足以容纳闭包时，会在堆上分配额外内存。用`std::function`声明闭包时会变慢，可能占用额外空间。

# Item1:模板类型推断

```cpp
template<typename T>
void f(ParamType param);
f(expr);
```

在编译期，编译器使用`expr`推断两类型:

- `T`
- `ParamType`:往往包含`const`、`&`

```cpp
template<typename T>
void f(const T& param); // ParamType is const T&
int x = 0;
f(x);
```

`f(x)`：`T`推断为`int`，`ParamType`推断为`const int&`

## 情况1：`ParamType`是引用或指针类型，但不是广义引用(&&)

- `expr`是引用时，**忽略其`&`**，然后与`ParamType`模式匹配来确定`T`,如：

```cpp
template<typename T>
void f(T& param);

int x = 27;
const int cx = x;
const int& rx = x;

f(x); // T is int, param's type is int&
f(cx);  // T is const int, param's type is const int&
f(rx);  // T is const int, param's type is const int&
```

推断期间，下面`rx`的`&`同样被忽略

```cpp
template<typename T>
void f(const T& param);

int x = 27;
const int cx = x;
const int& rx = x;

f(x); // T is int, param's type is int&
f(cx);  // T is int, param's type is const int&
f(rx);  // T is int, param's type is const int&
```

右值引用同理，但只有右值实参才会被赋给右值引用形参

- `expr`是指针(或pointer to const)时同理

```cpp
template<typename T>
void f(T* param);

int x = 27;
const int *px = &x;

f(&x);  // T is int, param's type is int*
f(px);  // T is const int, param's type is const int*
```

## 情况2：`ParamType`是广义引用

- `expr`是左值时，`T`和`ParamType`都推断为**左值引用**（这是在模板类型推断中，`T`被推断为引用的**唯一情况**）

```cpp
template<typename T>
void f(T&& param);

int x = 27;
const int cx = x;
const int& rx = x;

f(x); // x is lvalue, T is int&, param's type is int&
f(cx);  // cx is lvalue, T is const int&, param's type is const int&
f(rx);  // rx is lvalue, T is const int&, param's type is const int&
f(27);  // 27 is rvalue, T is int, param's type is int&&
```

- `expr`是右值时，使用**情况1**

## 情况3：`ParamType`非指针非引用

```cpp
template<typename T>
void f(T param);
```

`param`始终**按值传递参数**（Copy一个)，忽略`expr`类型中**实参本身**的`& const volatile`修饰词

```cpp
int x = 27;
const int cx = x;
const int& rx = x;

f(x); // T is int, param's type is int
f(cx);  // T is int, param's type is int
f(rx);  // T is in&, param's type is int
```

`const volatile`仅仅在**实参本身按值传递时**才被忽略（实参本身的修饰），当实参为指针或引用时，其所指向的对象的这些修饰不能忽略，如：

```cpp
template<typename T>
void f(T param);

const char* const ptr = "hello";
f(ptr); // ptr本身按值传递，修饰ptr的const忽略掉，而其指向的是const char*，指向对象的修饰不能被忽略，所以T is const char*
```

### 数组作为实参

数组和指针是**不同的类型**，但数组可退化为指针：

```cpp
const char name[] = "abc";  // name's type is const char [4]
const char * ptrToName = name; // ptrToName's type is const char *
```

当数组作为实参时，按值传递的模板`T`被推断为指针：

```cpp
template<typename T>
void f(T param);

f(name);  // T is const char *
```

按引用传递的模板`T`被推断为数组类型：

```cpp
template<typename T>
void f(T& param)

f(name);  // T is const char [4], param's type is const char (&)[4]

template<typename T, std::size_t N>
constexpr std::size_t arraySize(T (&)[N]) noexcept
{
    return N; // N 在编译期求出 为 4
}
```

### 函数作为实参

函数类型可退化为函数指针,

- 模板按值传递时，退化为指针
- 模板按引用传递时，不退化

```cpp
void someFunc(int, double);    // type is void(int,double)

template<typename T>
void f1(T param);

template<typename T>
void f2(T& param);

f1(someFunc)   // T is  void (*)(int, double)
f2(someFunc).  // T is  void (int, double)
```

## 总结

- 模板类型推断时，若实参是引用类型，则忽略其引用
- 推断按值传递的参数时，`const`和`volatile`被忽略
- 函数和数组在按值传递的模板中退化为指针

# Iterm2:auto 类型推断

`auto`类型推断就是模板类型推断，`auto`充当着模板推断中`T`的作用（区别：auto 将花括号括的数列当作std::initializer_list，但模板类型推断无法判断）

- case1: 类型修饰词是指针或引用，不是广义引用
- case2:类型修饰词时广义引用
- case3:类型修饰词不是指针或引用

```cpp
auto x = 2;              // case 3
// 等价的模板
template<typename T>
void func_for_x(T param);
func_for_x(2);           // param's type is x's type

const auto cx = x;       // case 3
// 等价的模板
template<typename T>
void func_for_cx(const T param);
func_for_cx(x);         // param's type is cx's type

const auto& rx = x;     // case 1
// 等价的模板
template<typename T>
void func_for_rx(const T& param);
func_for_rx(x);        // param's type is rx's type

// 广义引用
auto&& uref1 = x;      // x is lvalue, uref1's type is int&
auto&& uref2 = cx;     // cx is const lvalue, uref2's type is const int&
auto&& uref3 = 27;     // 27 is rvalue, uref3's type is int&&

// 使用 auto，数组和函数退化为指针
const char name[] = "abc";    // name' s type is const char[4];
auto arr1 = name;             // arr1's type is const char*
auto& arr2 = name;            // arr2's type is const char(&)[4]

void someFunc(int, double);
auto func1 = someFunc;        // func1's type is void(*)(int, double)
auto& func2 = someFunc;       // func2's type is void(&)(int, double)

// 唯一的例外
auto a = 27;
auto b(27);   // int

auto c = {27}; // std::initializer_list<int>
auto d{27};    // std::initializer_list<int>
```

当`auto`作为函数返回类型或 lambda 表达式的形参时，等价于**模板类型推断**而不是auto类型推断

# Item3:理解 decltype

取得表达式或名称的类型

```cpp
const int i = 0;         // decltype(i) is const int
bool f(const Widget& w); // decltype(w) is const Widget&
                         // decltype(f) is bool(const Widget&)
struct Point {
 int x, y;               // decltype(Point::x) is int
};                       // decltype(Point::y) is int
```

在函数模板中，返回值的类型依赖于参数的类型时(C++11)：

```cpp
template<typename Container, typename Index> // works, but
auto authAndAccess(Container& c, Index i)    // requires
 -> decltype(c[i])                           // refinement
{
	 authenticateUser();
	 return c[i];
}
```

C++14支持自动推断函数和lambda的返回类型：

```cpp
template<typename Container, typename Index> // C++14;
auto authAndAccess(Container& c, Index i)    // not quite
{                                            // correct
	 authenticateUser();
	 return c[i];                              // return type deduced from c[i]
}
```

但当`auto`应用于返回类型时，采用的是**模板类型推断方法**，此时引用和const会被忽略，对于`vector<int>` 来说，将返回`int` 类型，而我们期望的是`int&` 。

此时，我们需要使用 **`decltype`** 类型推导来指定其返回类型，即指定 **`authAndAccess`** 应该返回与表达式 **`c[i]`** 完全相同的类型。在 C++14 中，C++ 的设计者们为了应对在一些需要推断类型的情况下使用 **`decltype`** 类型推导规则的需求，引入了 **`decltype(auto)`** 说明符。auto 表明类型应该被推导，而 decltype 表示在推导期间应该使用 decltype 的规则：

```cpp
template<typename Container, typename Index> // C++14; works,
decltype(auto)                               // but still
authAndAccess(Container& c, Index i)         // requires
{                                            // refinement
	 authenticateUser();
	 return c[i];
}
```

在声明变量时：

```cpp
Widget w;
const Widget& cw = w;
auto myWidget1 = cw;           // auto type deduction: 忽略const、&, type is Widget
decltype(auto) myWidget2 = cw; // decltype type deduction: type is const Widget&
```

上面`authAndAccess` 的问题是：无法接收右值容器，解决办法是：

- 使用函数重载：左值引用和右值引用两个版本，太麻烦
- 使用广义引用参数

```cpp
// final C++14 version
template<typename Container, typename Index>
decltype(auto)
authAndAccess(Container&& c, Index i)
{
	 authenticateUser();
	 return std::forward<Container>(c)[i];
}

// final C++11 version
template<typename Container, typename Index>
auto // C++11
authAndAccess(Container&& c, Index i)
-> decltype(std::forward<Container>(c)[i])
{
	 authenticateUser();
	 return std::forward<Container>(c)[i];
}
```

特殊情况，将**`decltype`** 用于左值表达式时，其类型是`**&` ：**

```cpp
int x = 0;
// decltype(x) is int
// decltype((x)) is int&, x is lvalue
// decltype((x+1)) is int&&, x+1 is rvalue
```

# 推断类型的方法

```cpp
template<typename T> // 只有声明 TD 的类模板;
class TD; // TD == "Type Displayer"
```

尝试实例化这个模板会引发错误消息，因为没有模板定义可以实例化。为了查看 x 和 y 的类型，只需尝试用它们的类型实例化 TD：

```cpp
TD<decltype(x)> xType; // 引发包含 x 的类型的错误
TD<decltype(y)> yType; // 引发包含 y 的类型的错误
```

运行时打印类型：

```cpp
std::cout << typeid(x).name() << '\\\\\\\\n'; // display types for
std::cout << typeid(y).name() << '\\\\\\\\n'; // x and y

template<typename T>
void f(const T& param)
{
	  using std::cout;
		cout << "T = " << typeid(T).name() << '\\\\\\\\n';         // show T
		cout << "param = " << typeid(param).name() << '\\\\\\\\n'; // show param's type
    …
}
```

# Item5:使用auto声明类型

```cpp
template<typename It>
void dwim(It b, It e) {
    while(b != e) {
	    // 太长
        // typename std::iterator_traits<It>::value_type currValue = *b;
        auto currValue = *b;
        ...
    }
}
// function太麻烦
std::function<bool(const std::unique_ptr<Widget>&, const std::unique_ptr<Widget>&)>
derefUPLess
	= [](const std::unique_ptr<Widget>& p1, const std::unique_ptr<Widget>& p2)
    { return *p1 < *p2; };
```

C++14以后的 lambda：

```cpp
auto derefLess =
[](const auto& p1,const auto& p2) // values pointed
{ return *p1 < *p2; }; // to by anything
```

`auto` 声明的`derefLess` 就是闭包的类型，这与 `std::function`声明的类型不同，`std::function`类模板实例对象有一个固定的大小，当大小不足以容纳闭包时，会在堆上分配额外内存。用`std::function`声明闭包时会变慢，可能占用额外空间。

# Item6:当自动推断得到不希望的类型时，使用显式类型的初始化习惯用法

```cpp
std::vector<bool> features(const Widget& w);
bool highPriority = features(w)[5];
// auto highPriority = features(w)[5];   type is std::vector<bool>::reference
```

# Iterm7:Distinguish () and {} when creating objects

For user-defined types, it’s important to distinguish _initialization_ from _assignment_, because **different function calls are involved**:

```cpp
Widget w1;       // call default constructor
Widget w2 = w1;  // call copy constructor
w1 = w2;         // call copy operator=
```

C++11 introduces _uniform initialization_: **Braced initialization**

```cpp
std::vector<int> v{ 1, 3, 5 }; // v's initial content is 1, 3, 5
```

**`{}`** can also be used to specify default initialization values for **non-static data members:**

```cpp
class Widget {
 …
private:
 int x{ 0 }; // fine, x's default value is 0
 int y = 0;  // also fine
 int z(0);   // error!
};
```

**Uncopyable objects** may be initialized using **`{}`** or **`()`**, but not using **`=`**:

```cpp
std::atomic<int> ai1{ 0 }; // fine
std::atomic<int> ai2(0);   // fine
std::atomic<int> ai3 = 0;  // error!
```

It **prohibits** implicit _narrowing conversions_ among built-in types:

```cpp
double x, y, z;
…
int sum1{ x + y + z }; // error! sum of doubles may
                       // not be expressible as int

int sum2(x + y + z);   // okay (value of expression
                       // truncated to an int)
int sum3 = x + y + z; // ditto
```

It prohibits most vexing parse:

```cpp
Widget w1(10); // call Widget ctor with argument 10
Widget w2();   // most vexing parse! declares a function
               // named w2 that returns a Widget!
               // but want to call ctor with no args
Widget w3{}; // calls Widget ctor with no args
```

When constructors declare a parameter of type **`std::initializer_list`**, calls using the braced initialization syntax strongly prefer the overloads taking **`std::initializer_list` :**

```cpp
class Widget {
public:
	Widget(int i, bool b);   
	Widget(int i, double d);
	Widget(std::initializer_list<long double> il);
	operator float() const;  // convert
  …                        // to float
};

Widget w1(10, true); // uses parens and, as before,
                     // calls first ctor
Widget w2{10, true}; // uses braces, but now calls
                     // std::initializer_list ctor
                     // (10 and true convert to long double)
Widget w3(10, 5.0);  // uses parens and, as before,
                     // calls second ctor
Widget w4{10, 5.0};  // uses braces, but now calls
										 // std::initializer_list ctor
										 // (10 and 5.0 convert to long double)
Widget w5(w4);       // uses parens, calls copy ctor
Widget w6{w4};       // uses braces, calls
										 // std::initializer_list ctor
										 // (w4 converts to float, and float
										 // converts to long double)
Widget w7(std::move(w4)); // uses parens, calls move ctor
Widget w8{std::move(w4)}; // uses braces, calls
												  // std::initializer_list ctor
												  // (for same reason as w6)
```

# **Item 8: Prefer nullptr to 0 and NULL**

# **Item 9: Prefer alias declarations to typedefs**

- typedefs don’t support templatization, but alias declarations do.
- Alias templates avoid the “::type” suffix and, in templates, the “typename” prefix often required to refer to typedefs.
- C++14 offers alias templates for all the C++11 type traits transformations.

# **Item 10: Prefer scoped enums to unscoped enums**

scoped enums 是**强类型**的枚举，可以**指定底层数据类型**和**提前声明**：

```cpp
enum class H; // default is int
enum class Status: std::uint32_t { 
    good = 0,
    failed = 1,
    incomplete = 100,
    corrupt = 200,
    audited = 500,
    indeterminate = 0xFFFFFFFF
 };
```

unscoped enums 的方便之处：

```cpp
// unscoped enums
// name, email, reputation
using UserInfo = std::tuple<std::string, std::string, std::size_t>;
enum UserInfoFields { uiName, uiEmail, uiReputation };
UserInfo uInfo;
// get value of email field
// implicit conversion from UserInfoFields to std::size_t
auto val = std::get<uiEmail>(uinfo); 

// scoped enums
enum class UserInfoFields { uiName, uiEmail, uiReputation };
UserInfo uInfo;
// more verbose
auto val = std::get<static_cast<std::size_t>(UserInfoFields::uiEmail)>(uinfo)
```

**`std::get`** 在编译期计算结果，是**constexpr function。**可以通过编写一个函数接收enum返回一个**`std::size_t` 。**但当需要泛化返回类型时，可以使用**`std::underlying_type`** type trait。

```cpp
template<typename E>
constexpr auto
toUType(E enumerator) noexcept
{
	return static_cast<std::underlying_type_t<E>>(enumerator);
}
// get value
auto val = std::get<toUType(UserInfoFields::uiEmail)>(uInfo);
```

# Item 11:Prefer deleted functions to private undefined ones

**`istream`**和**`ostream`**为**`basic_ios`**的子类，为了防止复制它们，将**`basic_ios`**的**`copy ctor`**和**`copy operator =`** 设置为私有且未定义**：**

```cpp
template <class charT, class traits = char_traits<charT> >
class basic_ios : public ios_base {
public:
	…
private:
	basic_ios(const basic_ios&); // not defined
	basic_ios& operator=(const basic_ios&); // not defined
};
```

C++11的等价版本：

```cpp
template <class charT, class traits = char_traits<charT> >
class basic_ios : public ios_base {
public:
 …
 basic_ios(const basic_ios& ) = delete;
 basic_ios& operator=(const basic_ios&) = delete;
 …
};
```

Any function may be deleted, including non-member functions and template instantiations：

```cpp
template<typename T>
void processPointer(T* ptr);

template<>
void processPointer<void>(void*) = delete;
template<>
void processPointer<char>(char*) = delete;
```

# **Item 12: Declare overriding functions override**

For overriding to occur, several requirements must be met:

- The base class function must be **virtual**.
- The base and derived function **names** must be identical (except in the case of destructors).
- The **parameter types** must be identical.
- The **constness** must be identical.
- The **return types** and **exception specifications** must be compatible.
- The **reference qualifiers** must be identical.

```cpp
class Base {
public:
	virtual void mf1() const;
	virtual void mf2(int x);
	virtual void mf3()&;
	virtual void mf4() const;
};
class Derived : public Base {
public:
	virtual void mf1() const override;
	virtual void mf2(int x) override;
	virtual void mf3() & override;
	void mf4() const override; // adding "virtual" is OK, but not necessary
}; 
```

**函数引用限定符（reference qualifier）：** 限制成员函数仅用于左值对象或右值对象。

```cpp
class Widget {
public:
	using DataType = std::vector<double>;
	…
	DataType& data()& // for lvalue Widgets,
	{
		return values;  // return lvalue
	} 
	DataType data()&& // for rvalue Widgets,
	{
		return std::move(values); // return rvalue
	} 
	…
private:
	DataType values;
};

auto vals1 = w.data(); // calls lvalue overload for
											 // Widget::data, copy constructs vals1
auto vals2 = makeWidget().data();  // calls rvalue overload for
																	 // Widget::data, move constructs vals2 
```

# **Item 13: Prefer const_iterators to iterators**

C++11中：

```cpp
std::vector<int> values;
...
auto it = std::find(values.cbegin(), values.cend(), 1983);
values.insert(it, 1998);
```

C++11 对于`**const_iterator**`支持的唯一缺陷是只为`begin`和`end`提供了对应的非成员函数版本，而没有为`cbegin`、`cend`、`rbegin`、`cend`、`crbegin`和`crend`这些返回`const_iterator`的函数提供对应的非成员函数版本，这个问题在 C++14 中得到了解决。如下就是非成员函数版本的`cbegin`的一个实现方式：

```cpp
template<class C>
auto cbegin(const C& container) -> decltype(std::begin(container))
{
	return std::begin(container);
}
```

# **Item 14: Declare functions noexcept if they won’t emit exceptions**

apply **noexcept** to functions that won’t produce exceptions: it permits compilers to generate better object code.

在带有`noexcept`声明的函数中，优化器不需要在异常传出函数的前提下，将运行时栈保持在可展开状态；也不需要在异常逸出函数的前提下，保证所有其中的对象以其被构造顺序的逆序完成析构。而那些以`throw()`异常规格声明的函数就享受不到这样的优化灵活性，和那些没有加上异常规格的函数一样。`noexcept`属性对于移动操作、swap、内存释放函数和析构函数最有价值。C++11 STL 中的大部分函数遵循 “能移动则移动，必须复制才复制” 策略，但这必须保证在使用移动操作代替复制操作后，函数依旧具备强异常安全性。为了得知移动操作会不会产生异常，就需要校验这个操作是否带有`noexcept`声明。

标准库中的`swap`是否带有`noexcept`声明，取决于用户定义的`swap`自身。例如，标准库为数组和`std::pair`准备的`swap`函数如下：

```cpp
template<class T, size_t N>
void swap(T (&a)[N],
          T (&b)[N]) noexcept(noexcept(swap(*a, *b)));

template<class T1, class T2>
struct pair {
    ...
    void swap(pair& p) noexcept(noexcept(swap(first, p.first)) &&
                                noexcept(swap(second, p.second)));
    ...
};
```

这些函数带有条件式`noexcept`声明，它们到底是否具备`noexcept`属性，取决于它的`noexcept`分句中的表达式是否结果为`noexcept`。在此处，数组和`std::pair`的`swap`具备`noexcept`属性的前提是，其每一个元素的`swap`都具备`noexcept`属性。

在 C++11 中，内存释放函数和所有的析构函数都默认隐式地具备`noexcept`属性。析构函数未隐式地具备`noexcept`属性的唯一情况，就是所有类中有数据成员（包括继承而来的成员，以及在其他数据成员中包含的数据成员）的类型显式地将其析构函数声明为`noexcept(false)`，即可能抛出异常。

# **Item 15: Use constexpr whenever possible**

**Constexpr objects** are const, and have values that are known at compile time.

When **constexpr functions** are involved. Such functions produce compile-time constants _when they_

_are called with compile-time constants_. If they’re called with values not known until runtime, they produce runtime values.

- **constexpr functions** can be used in contexts that demand compile-time con‐ stants. If the values of the arguments you pass to a constexpr function in such a context are known during compilation, the result will be computed during compilation. If any of the arguments’ values is not known during compilation, your code will be rejected.
- When a constexpr function is called with one or more values that are not known during compilation, it acts like a normal function, computing its result at runtime. This means you don’t need two functions to perform the same opera‐ tion, one for compile-time constants and one for all other values. The constexpr function does it all.

```cpp
constexpr int pow(int base, int exp) noexcept {     // C++14
    auto result = 1;
    for (int i = 0; i < exp; ++i) result *= base;
    return result;
}
constexpr auto numConds = 5;
std::array<int, pow(3, numConds)> results; // results has
																					 // 3^numConds
																					 // elements
auto base = readFromDB("base");      // get these values
auto exp = readFromDB("exponent");   // at runtime
auto baseToExp = pow(base, exp);     // call pow function
                                     // at runtime
```

`constexpr`函数仅限于传入和返回**字面值类型（literal type）**，这些类型能够持有编译期可以决议的值。在 C++11 中，除了`void` (C++14包含）的所有内建类型都是字面类型；此外，我们也可以自定义字面类型，这需要将其构造函数和部分成员函数声明为`constexpr`函数：

```cpp
class Point {
public:
    constexpr Point(double xVal = 0, double yVal = 0) noexcept :
        x(xVal), y(yVal) {}

    constexpr double xValue() const noexcept { return x; }
    constexpr double yValue() const noexcept { return y; }
	  // c++ 14
	  // constexpr void setX(double newX) noexcept { x = newX; }
    void setX(double newX) noexcept { x = newX; }
    void setY(double newY) noexcept { y = newY; }

private:
    double x, y;
};

constexpr Point p1(9.4, 27.7);  // 在编译期执行 constexpr 构造函数
constexpr Point p2(28.8, 5.3);  // 同上

constexpr Point midpoint(const Point& p1, const Point& p2) noexcept {
    return { (p1.xValue() + p2.xValue()) / 2,
             (p1.yValue() + p2.yValue()) / 2 }; // 调用 constexpr 成员函数
}

constexpr auto mid = midpoint(p1, p2);  // 使用 constexpr 函数的返回值来初始化 constexpr 对象
```

# **Item 16: Make const member functions thread safe**