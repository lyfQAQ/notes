# The power of Move Semantics

```
std::vector<std::string> createAndInsert()
{
    std::vector<std::string> coll;
    coll.reserve(3);
    std::string s{"data"};
    coll.push_back(s);
    coll.push_back(s + s);
    coll.push_back(s);
    return coll;
}

std::vector<std::string> v;
v = createAndInsert();

```

本来`return coll`会调用拷贝构造来构造函数的返回值，但由于 _**named return value optimization(NRVO)**_，编译器生成相关代码，直接将`coll`当作返回值。

`v = createAndInsert()` 调用拷贝赋值：通过函数返回值(假设是`retval`)构造`v` ，这句代码执行完后，函数的返回值`retval` (其实是函数内部的`coll`)会析构掉，最后结果如图：


C++11之前上面的代码很多都是没必要的分配和释放。C++11 之后`s+s`的结果为 `rvalue`，会调用右值版本的`push_back()`，其内部改变 `coll[1]`指针指向，减少了一次深拷贝，如图：