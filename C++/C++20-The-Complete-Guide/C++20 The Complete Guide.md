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

```C++
//T 是浮点类型时，忽略该模板
template<typename Coll, typename T, typtename = std::enable_if<!std::is_floating_point_v<T>>>
void add(Coll& coll, const T& val)
{
  coll.push_back(val);
}
```

## Semantic Constraints

`concept` 会检查语法和语义约束：

- 语法约束：能在**编译期检查**对应的要求（如，操作是否支持，该操作是否产生对应的类型等）
- 语义约束：只能在**运行时检查**的要求（如，操作是否具有相同的效果，对于特定值执行的相同操作是否总是产生相同的结果）

### Example of Semantic Constrains

##### std::ranges::sized_range

该 `concept` 保证了一个 range 的包含的元素个数可在$O(1)$时间内计算出来（通过size()或计算 begin 和 end 的差）。

一个 `range` 类型提供了 `size()`时就默认满足这个 concept，如果要去掉限制，就需要设置`std::disable_sized_range<Rg> = true`：

```C++
class MyCont
{
  	...
    std::size_t size() const // 复杂度不是O(1)
}
// 需要解除该 concept
constexpr bool std::ranges::disable_sized_range<MyCont> = true;
```

##### std::ranges::range vs std::ranges::view

`std::ranges::view`除了一些语法约束外，它还保证了**移动构造函数/赋值**、**复制构造函数/赋值**（如果可用）和**析构函数**具有$O(1)$复杂度。

自定义`range`可以通过`public`继承 `std::ranges::view_base` 或` std::ranges::view_interface<> `，或者通过将模板特化 `std::ranges::enable_view<Rg>` 设置为 `true` 来提供相应的保证。

##### std::invocable vs std::regular_invocable

`std::invocable`保证了 it be called with `std::invoke`

```C++
template<typename F, typename... Args>
concept invocable = requires(F&& f, Args&&... args)
{
  	std::invoke(std::forward<F>(f), std::forward<Args>(args)...);
};

vod PrintVec(const std::vector<int>& vec, invocable<int> auto fn)
{
  	for(auto& elem : vec)
    		std::cout << fn(elem) << '\n';
}
std::vector<int> vec{1, 2, 3};
PrintVec(vec, [](int v) -> int { return -v; });
```

> `invocable<int> auto fn`此处`int`指定的是返回值类型第一个模板参数 `F`编译期通过 `fn`自动推断，无需添加。

`td::regular_invocable`添加了要求：必须是纯函数, 相同输入产生相同输出，这是**语义**上的要求，**编译期无法检查出来**。

```C++
// fn不是纯函数
auto fn = [i=0](int a) mutable { return a + ++i; };
// 编译期断言无法检查出来
static_assert(std::invocable<decltype(fn), int>);	// ok
static_assert(std::regular_invocable<decltype(fn), int>); // ok
// 但是连续两次调用fn会产生不同结果，这只能在运行时判断
fn(10) != fn(10) // true

```

##### std::weakly_incrementable vs. std::incrementable

- `incrementable` 要求相同值的每次递增都产生相同的结果。
- `weakly_incrementable` 仅要求类型支持递增运算符。递增相同的值可能会产生不同的结果。

因此：

- 当满足` incrementable `时，可以从**一个起始值多次迭代一个范围**。
- 当只满足 `weakly_incrementable` 时，您只能**对一个范围进行一次迭代**。**使用相同的起始值进行第二次迭代可能会产生不同的结果**。

**输入流迭代器**只能迭代一次，下次迭代会产生不同的值，满足`weakly_incrementable` 。这都是语义上的限制，只能通过**不同的函数签名来区分**：

```C++
std::weakly_incrementable<std::istream_iterator<int>> // 返回 true 
std::incrementable<std::istream_iterator<int>> // 也返回 true 但语义上是 false 编译期无法判断
  
// single-pass algorithm
// 可传递流迭代器
template<std::weakly_incrementable T> 
void algo1(T beg, T end); 

// multi-pass algorithm 
// 不能传递流迭代器给该函数
template<std::incrementable T> 
void algo2(T beg, T end);

algo1(std::istream_iterator<int>{std::cin}, std::istream_iterator<int>{}); // Ok
algo2(std::istream_iterator<int>{std::cin}, std::istream_iterator<int>{}); // Bad

```

不能通过**编译期的 **`concept` 来区分，下面的写法就是错的，因为语法上无法区分二者：

```C++
template<std::weakly_incrementable T> 
void algo(T beg, T end);

template<std::incrementable T> 
void algo(T beg, T end);

```

对于这种语义上的差异，可使用`iterator concepts`来区分：

```C++
template<std::input_iterator T> 
void algo(T beg, T end);	// single-pass algorithm
template<std::forward_iterator T> 
void algo(T beg, T end);  // multi-pass algorithm  
// Ok: overload resolution select the single-pass algorithm
algo(std::istream_iterator<int>{std::cin}, std::istream_iterator<int>{});
```

## Design Guidelines for Concepts

`concept`是 subsumption 的，意味着一个 `concept` 可以是另一个 `concept` 的子集。优先使用 `concept` 而不是`type traits`、编译期表达式等。

```C++
template<typename T, typename U> 
requires std::is_same_v<T, U> // using traits 
void foo(T, U) 
{ 
  	std::cout << "foo() for parameters of same type" << '\n'; 
}

template<typename T, typename U> 
requires std::is_same_v<T, U> && std::is_integral_v<T> 
void foo(T, U) 
{ 
  	std::cout << "foo() for integral parameters of same type" << '\n'; 
}

foo(1, 2); // ERROR: ambiguity: both requirements are true
```

两个 requires 表达式都为 true，重载决议无法得到它们的优先级。

使用 `concept`，编译器会优先选择第二个版本：

```C++
template<typename T, typename U> 
requires std::same_as<T, U> // using concepts 
void foo(T, U) 
{ 
  	std::cout << "foo() for parameters of same type" << '\n'; 
}

template<typename T, typename U> 
requires std::same_as<T, U> && std::integral<T> 
void foo(T, U) 
{ 
  	std::cout << "foo() for integral parameters of same type" << '\n'; 
}

foo(1, 2); // OK: second foo() preferred
```

# Concepts, Requirements, and Constraints in Detail

## Constraints

为了指定泛型参数的requirements，需要使用constraints来在编译期决定是否实例化模板。可以constrain:

- function templates
- class templates
- variable templates
- alias templates

使用`requires`子句来定义constraint：

```C++
template<typename T>
void foo(const T& arg) requires MyConcept<T>
{...}
```

也能在模板参数或 `auto` 前面使用`concept`来为类型添加 constraint：

```C++
template<MyConcept T>
void foo(const T& arg)
{...}

// same use auto
void foo(const MyConcept auto& arg)
{...}
```

## requires Clauses

`requires` 子使用 `requires` 关键字和编译期 `bool `表达式来限制模板，bool 表达式可以是：

- Ad-hoc bool 表达式
- concept
- requires expression

使用 bool 表达式的地方，也可以使用 constraints 

### Using && and || in requires Clauses

在 requires 子句中组合多个 constraints：

```C++
template<typename T>
requires (sizeof(T) > 4)		// 编译期 bool 表达式
					&& requires { typename T::value_type;}	// requires 表达式
					&& std::input_iterator<T>	// concept
void foo(T x) {...}

template<typename T>
requires std::integral<T> || std::floating_point<T>
T power(T b, T p);
```

## Ad-hoc Boolean Expressions

requires子句中可包含的表达式有：

- Type predicates, such as type traits
- Compile-time variables (defined with constexpr or constinit)
- Compile-time functions (defined with constexpr or consteval)

```C++
template<typename T> 
requires (sizeof(int) != sizeof(long)) 
...

template<typename T, std::size_t Sz> 
requires (Sz > 0) 
...

template<typename T> 
constexpr bool gcd(T a, T b); // greatest common divisor (forward declaration)

template<typename T, int Min, int Max> 
requires (gcd(Min, Max) > 1) // available if there is a GCD greater than 1 
...
```



## requires Expressions

requires表达式提供了更灵活的方法，对一个或多个模板参数指定多个要求，可以是：

- Required type definitions

- Expressions that have to be valid

- Requirements on the types that expressions yield

```C++
template<typename Coll>
... requires {
  typename Coll::value_type::first_type;		// elements have first_type
  typename Coll::value_type::second_type;		// elements have second_type
}
```

可选参数列表，引入了**虚拟参数**来表达 requirements：

```C++
template<typename T>
...
requires(T x, T y)
{
  	x + y;	// 支持+
  	x - y;	// 支持-
}
```

虚拟参数永远不会被实参替换，所以声明为值还是引用并不影响。

虚拟参数类型还可以依赖模板参数类型：

```C++
template<typename Coll>
...
requires(Coll::value_type v)	// 检查Call::value_type是否有效，此处不必使用typename前缀
{
 		st::cout << v;		// 支持输出
}

// 只检查是否有value_type
template<typename Coll>
...
requires(Coll::value_type v)	// 检查Call::value_type是否有效
{
 		true;   // 花括号内不能为空，所以写个 ture 就行
}
```

### Simple Requirements

形式良好的表达式，如果调用它们必须保证编译通过。但实际上并不会执行，所以结果不重要：

```C++
template<typename T1, typename T2>
...
requires(T1 val, T2 p)
{
  	*p;						// T2支持 operator*
  	p[0];					// 支持operator[int]
  	p->value();		// 具有成员函数 value()
  	*p > val;			// 支持 operator*的结果与 val 比较
  	p == nullptr	// 支持与 nullptr 比较
}
```

> `p == nullptr`不是要求 p 是空指针
>
> `*p > val || p == nullptr`表示可以用||组合两个子表达式的结果

如果要求支持两个子表达式中的任意一个，必须：

```C++
template<typename T1, typename T2>
...
requires(T1 val, T2 p)
{
  	*p > val;	  // 支持 operator* 的结果与 T1 的比较 
}
||	// 或者
requires(T2 p)
{
  	p == nullptr;// 支持 T2 与 nullptr 的比较 
}
```

注意，下面的 `concept`并不是要求 `T` 是整数类型：

```C++
template<typename T>
...
requires
{
  	std::integral<T>;	//要求该表达式是合法的（其实对所有类型T都合法）
}
```

如果要求 `T `是整数类型，则要这样：

```C++
template<typename T> 
... 
std::integral<T> && // 正确，要求 T 是整数类型
requires 
{ 
    ...
};

//或者
template<typename T> 
... 
requires 
{ 
    requires std::integral<T>; // 正确，要求 T 是整数类型
    ... 
};


```

### Type Requirements

保证类型名称是有效的：

```C++
template<typename T1, typename T2>
...
requires
{
  	typename T1::value_type; 							// T1 需要有成员类型 value_type 
    typename std::ranges::iterator_t<T1>; // T1 需要有迭代器类型 
    typename std::common_type_t<T1, T2>; 	// T1 和 T2 需要有公共类型 
}

// 实例
template<std::integral T> 
class MyType1 
{ 
    //...
};
template<typename T> 
requires requires 
{ 
    typename MyType1<T>; // instantiation of MyType1 for T would be valid
} 
void mytype1(T) 
{ 
    ...
}
// OK
mytype1(42); 
// ERROR
mytype1(7.7);

```

如果 `T`是 `void`，则**满足任何的类型要求**。

下面的 `concept` 并不是检查类型 `T` 是否存在 `hash` 函数：

```C++
template<typename T> 
concept StdHash = requires 
{ 
    typename std::hash<T>; // 不检查 std::hash<> 是否为类型 T 定义 
};
```

应该尝试创建对象：

```C++
template<typename T> 
concept StdHash = requires 
{ 
    std::hash<T>{}; // OK，检查是否可以为类型 T 创建一个标准哈希函数 
};
```

使用产生值或产生类型的类型函数是无意义的，因为它总能产生一个值或类型，永远是合法的：

```C++
template<typename T> 
... 
requires 
{ 
    std::is_const_v<T>; // 没用：总是合法的（值是什么无关紧要）
}

template<typename T> 
... 
requires 
{ 
    typename std::remove_const_t<T>; // 没用：总是合法的（总是会产生一个类型）
}
// 正确做法
// 应该放在花括号外面
template<typename T> 
... std::is_const_v<T>	// ok.  ensure T is const

```

使用可能具有未定义行为的类型函数也是没有意义的。例如，type traits `std::make_unsigned<>` 要求传递的参数是除了 `bool` 之外的整数类型。如果传递的类型不是整数类型，则会导致未定义行为：

```C++
template<typename T> 
... 
requires 
{ 
    std::make_unsigned<T>::type; 	// 有效或未定义行为
}
// 正确做法
// 嵌套requires
template<typename T> 
... 
requires 
{ 
    requires (std::integral<T> && !std::same_as<T, bool>); 
    std::make_unsigned<T>::type; // OK 
}


```

