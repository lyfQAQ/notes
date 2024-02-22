# constexpr and static constexpr

## constexpr

下面代码可以编译：

```cpp
  struct S {
  constexpr S(int x, int y): xVal(x), yVal(y) {}  
  constexpr S(int x): xVal(x) {}  
  constexpr S() {}
  const int xVal { 0 };          // ok!  const int yVal { 0 };          // ok!};
```

而下面不能编译：

```cpp
  struct S {
  constexpr S(int x, int y): xVal(x), yVal(y) {}  
  constexpr S(int x): xVal(x) {}  
  constexpr S() {}
  constexpr int xVal { 0 };         // error!  constexpr int yVal { 0 };         // error!};
```

- `constexpr` 表示可以在**编译期求值**
- 类的成员变量是在**运行时**调用构造函数初始化的，对象在初始化后它才存在
- 包含**xVal**成员的类对象可以是**constexpr**，这将导致**xVal**成为**constexpr**，如下

```c
struct S {
    constexpr S(int x, int y) : xVal(x), yVal(y) {}    constexpr S(int x) : xVal(x) {}    constexpr S() {}
    int xVal{ 0 };    int yVal{ 0 };};

template<int x>                   // x requires constexpr int foo() { return x; }

int main() {
    constexpr S s;    std::cout << foo<s.xVal>();       // ok!}
```

## Fibonacci

```c
template<int I>
struct Fib
{
    static const int val = Fib<I - 1>::val + Fib<I - 2>::val;};

template<>
struct Fib<0>
{
    static const int val = 0;};

template<>
struct Fib<1>
{
    static const int val = 1;};

int main() {
    std::cout << Fib<45>::val <<'\\\\n';}
```

# std::next

```cpp
std::vector<int> v{ 3, 1, 2, 3, 4, 5 };
std::cout << std::is_sorted(std::begin(v) + 1, std::end(v)) << '\\\\n';       // 随机迭代器才支持+1 -1
std::list<int> l{ 3, 1, 2, 3, 4, 5 };
// std::cout << std::is_sorted(std::begin(l) +1, std::end(l)) << '\\\\n';     // error!  ，list是双向迭代器
std::cout << std::is_sorted(std::next(std::begin(l)), std::end(l)) << '\\\\n';
```

# std::exchange

用法：

```cpp
delete std::exchange(ptr, nullptr);
```

等价于：

```cpp
delete ptr;
ptr = nullptr;
```

# **constexpr if**

Select branch at compile time:

```cpp
template<typename T>
auto print_type_info(const T& t)
{
	if constexpr(std::is_integral_v<T> && !std::is_same_v<bool, T>) {
		return t + 1;
	}
	else if constexpr (std::is_floating_point_v<T>) {
		return t + 0.1;
	}
	else {
		return t;
	}
}
std::cout << print_type_info(5) << '\\n';        // 6
std::cout << print_type_info(5.0) << '\\n';      // 5.1
std::cout << print_type_info(true) << '\\n';     // 1
```

# fold expression

```cpp
template<typename... T>
auto avg(T... t)
{
	return (t + ...) / sizeof...(t);
}
std::cout << avg(1, 2, 3.0, 4.5);
```

# Coroutine