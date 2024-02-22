# C++20 The Complete Guide

# Concepts,Requirements,and Constraints

## Using a requires Clause

进一步限制模板参数，T 不能是指针类型：

```cpp
template<typename T>
requires (!std::is_pointer_v<T>)
T maxValue(T a, T b)
{
    return b < a ? a : b;
}
```

### Defining a concept

concept就是对限制规则起个名字，方便多个地方使用：

```cpp
template <typename T>
concept IsPointer = std::is_pointer_v<T>;

template <typename T>
    requires(!IsPointer<T>)
T maxValue(T a, T b)
{
    return b < a ? a : b;
}
```

### Overloading with Concepts

使用 concept 来重载函数，

```cpp
template <typename T>
    requires(!IsPointer<T>)// need ()T maxValue(T a, T b)
{
    return b < a ? a : b;
}

template <typename T>
    requires IsPointer<T>// for pointersauto maxValue(T a, T b)
{
    return maxValue(*a, *b);
}
```

> 注：否定的 concept 需要在 requires 字句加括号

### Overload Resolution with Concepts

重载决议认为具有 concept 的模板比一般的模板更为特殊（**更优先**考虑使用）

### Type Constraints

当约束条件仅用在一个模板参数上时，可直接作为类型限制来声明模板参数：

```cpp
template <IsPointer T>// only for pointerauto maxValue(T a, T b)
{
    return maxValue(*a, *b);
}
```

使用 auto 声明参数时，也可将其作为类型限制：

```cpp
auto maxValue(IsPointer auto a, IsPointer auto b)
{
    return maxValue(*a, *b);
}
```

多个类型参数：

```cpp
template <IsPointer T1, IsPointer T2>
auto maxValue(T1 a, T2 b)
{
    return maxValue(*a, *b);
}
```

### Trailing requires Clauses

下面的代码还需要限制条件：指针指向的对象应该是可比较的类型

```cpp
auto maxValue(IsPointer auto a, IsPointer auto b)
{
    return maxValue(*a, *b);// need cmp
}
// 在函数参数列表后，使用尾置requires子句// std::totally_ordered_with 检查两种类型是否支持所有比较运算auto maxValue(IsPointer auto a, IsPointer auto b)
    requires std::totally_ordered_with<decltype(*a), decltype(*b)>
{
    return maxValue(*a, *b);
}
```

### requires Expression

但上述`maxValue()`模板并不支持**智能指针**类型，通过 **requires expression** 指定对象必须支持的操作：

```cpp
template <typename T>
concept IsPointer = requires(T p) { *p; };// *p has to be well-formed
```

这要求T类型支持`operator*`,即表达式`*p`必须合法

下面的 requires expression 制定了三个限制：

- 该类型支持operator*
- 该类型支持`operator<`，其返回值是 `bool` 类型
- 可以和`nullptr`比较

```cpp
template <typename T>
concept IsPointer = requires(T p)
{
    *p;// operator* has to be valid
    p == nullptr;// can compare with nullptr
    {p < p} -> std::same_as<bool>;// operator < yields bool
};
```

> 注：指定表达式产生的结果类型不能简单的写成 -> bool，必须使用 -> std::same_as<bool>
>
> 这些约束条件都是在编译期使用的，对生成的代码无影响

也可以在 requires 子句中使用 requires expression：

```cpp
template <typename T>
requires requires(T p) { *p;}
auto maxValue(T a, T b)
{
    return maxValue(*a, *b);
}
```

## **Use Constraints and Concepts**

```cpp
// fn template
template <typename T>
requires ...
void print(const T&) {...}

// class template
template <typename T>
requires ...
class MyType {...}
//还有以下
// Alias template
// variable template
// member fn
```

但**不能对 concept 进行约束**，如：

```cpp
template<typename T>
concept A = expr_1;

template<**A T**>       // 麻烦，还要跳到 A 的定义看看详情
concept B = expr_2;   // error

template<**typename T**>
concept B = A<T> && expr_2;   // ok 简单，直接能看出 B 的内容
```

因为 concept 对模板参数定义了一些约束，其所有内容在`=`之后，为了方便只需查看`=` 后的内容，而不必跳转查看其他的定义。

### Constraining Alias Templates

```cpp
template <std::ranges::range T>
using ValueType = std::ranges::range_value_t<T>;//T 中元素类型
// 等价形式
template <typename T>
requires std::ranges::range<T>
using ValueType = std::ranges::range_value_t<T>;

ValueType<int> vt1;              // error
ValueType<std::vector<int>> vt2; // int
ValueType<std::vector<double>> vt3; // double
```

### Constraining Variable Templates

```cpp
template <std::ranges::range T>
constexpr bool IsIntegralValType = std::integral<std::ranges::range_value_t<T>>;
// 等价形式
template <typename T>
requires std::ranges::range<T>
constexpr bool IsIntergralValType = std::integral<std::ranges::range_value_t<T>>;

bool b1 = IsIntegralValType<int>;   // ERROR
bool b2 = IsIntegralValType<std::vector<int>>; // true
```

### Constraining Member Functions

```cpp
template <typename T>
class ValOrColl
{
    T value;
public:
    ValOrColl(const T& val): value{val} {}
    ValOrColl(T&& val): value{std::move(val)} {}
    // 其他类型
    void print() const 
    {
        std::cout << value << '\\n';
    }
		// 对应集合类型
    void print() const requires std::ranges::range<T>   
    {
        for (const auto& elem: value)
        {
            std::cout << elem << ' ';
        }
        std::cout << '\\n';
    }
};
```

- 如果 `T`是集合类型，两个 `print` 函数都是合法的，但**重载决议会优先选择受约束限制的版本**
- 如果 `T` 不是集合类型，只有第一个 `print` 合法

限制具体类型是非法的：

```cpp
void foo() requires std::numeric_limits<char>::is_signed // ERROR
{
    ...
}

```

### Constraining Non-Type Template Parameters

```cpp
template <int Val> // or template <auto Val>
concept LessThan10 = Val < 10;

template <typename T, int Size>
requires LessThan10<Size>
class MyType{...};
```

## Typical Applications of Concepts

### Using Concepts to Understand Code and Error Messages

```cpp
template <typename Coll, typename T>
void add(Coll& coll, const T& val)
{
    coll.push_back(val);
}
```

上述代码存在隐含要求：

- `Coll` 类型支持`push_back`
- `T` 可以转换成 `Coll` 内部元素类型
- `T` 支持复制

```cpp
std::vector<int> vec;
add(vec, 42); // OK
add(vec, "hello"); // ERROR: no conversion from string literal to int

std::set<int> coll;
add(coll, 42); // ERROR: no push_back() supported by std::set<>

std::vector<std::atomic<int>> aiVec;
std::atomic<int> ai{42};
add(aiVec, ai); // ERROR: cannot copy/move atomics
```

定义 concept 增强编译错误可读性：

```cpp
// 支持 push_back
template <typename Coll, typename T>
concept SupportPushBack = requires(Coll c, T v)
{
    c.push_back(v);
};

template <typename Coll, typename T>
requires SupportPushBack<Coll, T>
void add(Coll& coll, const T& val)
{
    coll.push_back(val);
}

std::set<int> vec;
add(vec, 42); // ERROR

/opt/homebrew/opt/llvm/bin/clang++   -g -std=gnu++2b -arch arm64 -isysroot /Library/Developer/CommandLineTools/SDKs/MacOSX14.2.sdk -fcolor-diagnostics -MD -MT CMakeFiles/algo.dir/main.cpp.o -MF CMakeFiles/algo.dir/main.cpp.o.d -o CMakeFiles/algo.dir/main.cpp.o -c /Users/lyf/Desktop/work/algo/main.cpp
/Users/lyf/Desktop/work/algo/main.cpp:22:5: error: no matching function for call to 'add'
   22 |     add(vec, 42);
      |     ^~~
/Users/lyf/Desktop/work/algo/main.cpp:14:6: note: candidate template ignored: constraints not satisfied [with Coll = std::set<int>, T = int]
   14 | void add(Coll& coll, const T& val)
      |      ^
/Users/lyf/Desktop/work/algo/main.cpp:13:10: note: because 'SupportPushBack<std::set<int>, int>' evaluated to false
   13 | requires SupportPushBack<Coll, T>
      |          ^
/Users/lyf/Desktop/work/algo/main.cpp:9:7: note: because 'c.push_back(v)' would be invalid: no member named 'push_back' in 'std::set<int>'
    9 |     c.push_back(v);
      |       ^
1 error generated.
```

```cpp
// 支持类型转换
template <typename Coll, typename T>
requires std::convertible_to<T, typename Coll::value_type>
void add(Coll& coll, const T& val)
{
    coll.push_back(val);
}
std::vector<int> vec;
add(vec, "haha");

/opt/homebrew/opt/llvm/bin/clang++   -g -std=gnu++2b -arch arm64 -isysroot /Library/Developer/CommandLineTools/SDKs/MacOSX14.2.sdk -fcolor-diagnostics -MD -MT CMakeFiles/algo.dir/main.cpp.o -MF CMakeFiles/algo.dir/main.cpp.o.d -o CMakeFiles/algo.dir/main.cpp.o -c /Users/lyf/Desktop/work/algo/main.cpp
/Users/lyf/Desktop/work/algo/main.cpp:18:5: error: no matching function for call to 'add'
   18 |     add(vec, "haha");
      |     ^~~
/Users/lyf/Desktop/work/algo/main.cpp:10:6: note: candidate template ignored: constraints not satisfied [with Coll = std::vector<int>, T = char[5]]
   10 | void add(Coll& coll, const T& val)
      |      ^
/Users/lyf/Desktop/work/algo/main.cpp:9:10: note: because 'std::convertible_to<char[5], typename vector<int>::value_type>' evaluated to false
    9 | requires std::convertible_to<T, typename Coll::value_type>
      |          ^
/opt/homebrew/opt/llvm/bin/../include/c++/v1/__concepts/convertible_to.h:27:26: note: because 'is_convertible_v<char[5], int>' evaluated to false
   27 | concept convertible_to = is_convertible_v<_From, _To> && requires { static_cast<_To>(std::declval<_From>()); };
      |                          ^
1 error generated.
```

### Using Concepts to Disable Generic Code

考虑上述 `add` 函数需要对浮点类型做特殊处理：

```cpp
template <typename Coll, typename T>
void add(Coll& coll, const T& val)
{
    coll.push_back(val);
}
// for double
template <typename Coll>
void add(Coll& coll, double val)
{
		// special code for double...
    coll.push_back(val);
}

std::vector<int> iVec;
add(iVec, 42); // OK: calls add() for T being int
std::vector<double> dVec;
add(dVec, 0.7); // OK: calls 2nd add() for double
float f = 0.7;
add(dVec, f); // OOPS: calls 1st add() for T being float
```

`add(dVec, f)`采用第一个`add`函数，此处[[重载决议]]的规则：

- **无类型转换的调用**优先于有类型转换的调用
- **普通函数**优先于函数模板

调用第一个 `add` 函数无类型转换，所以被采纳。所以不应该使用具体类型声明第二个参数，应该用 concept 来限制 `T` ：

```cpp
template <typename Coll, typename T>
void add(Coll& coll, const T& val)
{
    coll.push_back(val);
}

// second add function
template<typename Coll, typename T>
requires std::floating_point<T>
void add(Coll& coll, const T& val)
{
		... // special code for floating-point values
		coll.push_back(val);
}
// same
template<typename Coll, std::floating_point T>
void add(Coll& coll, const T& val)
{
		... // special code for floating-point values
		coll.push_back(val);
}
// same
void add(auto& coll, const std::floating_point auto& val)
{
		... // special code for floating-point values
		coll.push_back(val);
}
```

要注意的是，函数的签名差异不应该过大，否则重载决议无法工作。

### Restrictions on Narrow

```cpp
std::vector<int> iVec;
add(iVec, 1.9); // 内部发生隐式类型转换，将会 1.9 截断成 1
```

通过下面的 concept 保证了可以安全的转换而不发生narrowing：

```cpp
template<typename From, typename To>  
concept ConvertsWithoutNarrowing =  
        std::convertible_to<From, To> &&  
        requires (From&& x)  
        {  
            {std::type_identity_t<To[]>{std::forward<From>(x)}} -> std::same_as<To[1]>;  
        };  
  
template<typename Coll, typename T>  
requires ConvertsWithoutNarrowing<T, typename Coll::value_type>  
void add(Coll& coll, const T& val)  
{  
    coll.push_back(val);  
}
```

### Using Requiresments to Call Different Functions

### Using Concept to Call Different Functions

```cpp
// 支持 push_back操作
template<typename Coll, typename T>
concept SupportsPushBack = requires(Coll c, T v)
{
	c.push_back(v);
};

// declval<>() to get a value of the element type
template<typname Coll>
concept SupportsPushBack = requires(Coll coll)
{
	 // 加了& 要求 使用 copy 版本的push_back，不加的话要求 move版本
	coll.push_back(std::declval<typename Coll::value_type&>());
};

// std::ranges::range_value_t
template<typename Coll>
concept SupportsPushBack = requires(Coll coll)
{
	coll.push_back(declval<std::ranges::range_value_t<Coll>>());
};
```
重载两种版本：
```cpp
template<typename Coll, typename T> 
requires SupportsPushBack<Coll> 
void add(Coll& coll, const T& val) 
{ 
	coll.push_back(val);
}

template<typename Coll, typename T> 
void add(Coll& coll, const T& val) 
{ 
	coll.insert(val);
}
```
### Concepts and requires for if constexpr
在 `if constexpr` 中使用 concept 和 requres

```cpp
// concept
if constexpr (SupportsPushBack<delctype(coll)>) 
{
	coll.push_back(val);
}
else
{
	coll.insert(val);
}
// requires 等价写法，更方便，不用定义 concept
if constexpr (requires {coll.push_back(val);})
{
	coll.push_back(val);
}
else
{
	coll.insert(val);
}
```
### Inserting Single and Multiple Values
使用`std::ranges::input_range`来让 `add` 支持添加多个值，该 concept 表示支持`std::ranges::begin()`和`std::ranges::end()`的集合：
```cpp
// 更加特殊，重载决议会优先考虑该模板
template<SupportsPushBack Coll, std::ranges::input_range T>
void add(Coll& coll, const T& val)
{
	coll.insert(coll.end(), std::ranges::begin(val), std::ranges::end(val));
}
// 一般
template<typename Coll, std::ranges::input_range T>
void add(Coll& coll, const T& val)
{
	coll.insert(std::ranges::begin(val), std::ranges::end(val));
}
```

### Dealing with Multiple Constraints
组合多个 concept 和 requires, [[`std::ranges::range_value_t`]]获取集合中元素的类型：
```cpp
template<SupportsPushBack Coll, std::ranges::input_range T>
requires ConvertsWithoutNarrowing<std::ranges::range_value_t<T>, typename Coll::value_type>
void add(Coll& coll, const T& val)
{
	coll.insert(std::ranges::begin(val), std::ranges::end(val));
}
```
等价写法，将所有 concept 放到一个 requires 子句中：
```cpp
template<typename Coll, typename T>
requires SupportsPushBack<Coll> && std::ranges::input_range<T> && 
			ConvertsWithoutNarrowing<std::ranges::range_value_t<T>, typename Coll::value_type>
void add(Coll& coll, const T& val)
{
	coll.insert(std::ranges::begin(val), std::ranges::end(val));
}
```
## 之前的禁用模板的方法
#### SFINAE
C++20之前使用 SFINAE禁用模板，如果模板声明的形式是错误的，那么就忽略该模板而不产生编译错误：
```cpp
// push_back作为模板声明的一部分，如果不支持该操作，救护忽略该模板
template<typename Coll, typename T>
auto add(Coll& coll, const T& val) -> decltype(coll.push_back(val))
{
	return coll.push_back(val);
}

template<typename Coll, typename T>
auto add(Coll& coll, const T& val) -> decltype(coll.insert(val))
{
	return coll.insert(val);
}
```
#### std::enable_if<>
适用于更复杂的情况，是一个type trait，接受一个布尔条件，如果条件为假，则产生无效代码，导致忽略该模板（被SFINAE掉）

