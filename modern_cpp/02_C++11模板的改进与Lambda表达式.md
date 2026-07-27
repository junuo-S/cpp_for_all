# 02 C++11模板的改进与Lambda表达式

C++11 对模板系统进行了多项实用改进，同时引入了 Lambda 表达式这一革命性语法。模板改进让泛型编程更加灵活，Lambda 则让"就地定义匿名函数"成为现实。两者结合，使 C++ 的抽象能力和表达力跃上了新台阶。

---

## 一、模板语法改进

### 1.1 右尖括号 >> 的修正

C++11 之前，嵌套模板的右尖括号之间必须加空格，否则编译器会把 `>>` 解析为右移运算符：

```cpp
// C++03：必须加空格
vector<vector<int> > matrix;  // 注意 > > 之间的空格

// C++11：直接写即可
vector<vector<int>> matrix;   // 编译器能正确解析
map<string, vector<int>> data;
```

这虽然只是一个小改动，却消除了一个长期困扰初学者的"语法陷阱"。

### 1.2 模板别名 using

C++11 之前，我们用 `typedef` 给类型起别名，但 `typedef` **无法直接给模板起别名**：

```cpp
// C++03 的笨办法：包一层结构体（假设 MyAllocator 已定义）
template <typename T>
struct MyVec {
    typedef vector<T, MyAllocator<T>> type;
};

MyVec<int>::type v;  // 使用繁琐
```

C++11 引入了 `using` 别名声明，可以直接定义**模板别名**（Alias Template）：

```cpp
#include <iostream>
#include <vector>
#include <map>
#include <string>
using namespace std;

// 模板别名：简洁直观
template <typename T>
using MyVec = vector<T>;

template <typename K, typename V>
using MyMap = map<K, V>;

// 非模板类型别名也可以用 using（替代 typedef）
using IntVec = vector<int>;
using StrIntMap = map<string, int>;

int main() {
    MyVec<int> v{1, 2, 3};
    MyMap<string, int> m{{"age", 25}, {"score", 90}};
    IntVec v2{4, 5, 6};

    for (int x : v) cout << x << " ";
    cout << endl;

    for (const auto& [k, val] : m) cout << k << ": " << val << endl;

    return 0;
}
```

| 对比项 | `typedef` | `using` |
|--------|-----------|---------|
| 基本别名 | `typedef int MyInt;` | `using MyInt = int;` |
| 模板别名 | 不支持（需包装） | `template<T> using Vec = vector<T>;` |
| 可读性 | 类型名在中间 | 类型名在左侧，类似赋值 |

### 1.3 extern template 显式实例化声明

在大型项目中，同一个模板（如 `vector<int>`）可能在多个编译单元中被实例化，导致重复编译、增大目标文件体积。C++11 的 `extern template` 可以告诉编译器：**本文件不要实例化这个模板，链接时去别处找**。

```cpp
// utils.h
#include <vector>
using namespace std;

extern template class vector<int>;  // 声明：不在本编译单元实例化

// utils.cpp
#include "utils.h"
template class vector<int>;  // 定义：在此处显式实例化（只编译一次）
```

这类似于 `extern` 变量的思路，可以显著减少编译时间和目标文件大小。

### 1.4 函数模板的默认模板参数

C++11 之前，只有类模板可以有默认模板参数。C++11 将这一能力扩展到了函数模板：

```cpp
#include <iostream>
using namespace std;

template <typename T = int, typename U = double>
T Convert(U value) {
    return static_cast<T>(value);
}

int main() {
    auto a = Convert(3.14);          // T=int, U=double → 3
    auto b = Convert<float>(3.14);   // T=float, U=double → 3.14f
    auto c = Convert<long, int>(42); // T=long, U=int → 42L

    cout << a << " " << b << " " << c << endl;

    return 0;
}
```

---

## 二、可变参数模板

### 2.1 什么是参数包

C++11 允许模板接受**任意数量、任意类型**的参数，这就是可变参数模板（Variadic Templates）。参数用省略号 `...` 表示：

```cpp
template <typename... Args>
void Print(Args... args);
```

这里 `Args` 是一个**模板参数包**（Template Parameter Pack），`args` 是一个**函数参数包**（Function Parameter Pack）。

### 2.2 sizeof... 运算符

`sizeof...` 可以获取参数包中参数的个数：

```cpp
#include <iostream>
using namespace std;

template <typename... Args>
void ShowCount(Args... args) {
    cout << "参数个数: " << sizeof...(Args) << endl;   // 类型个数
    cout << "参数个数: " << sizeof...(args) << endl;   // 值个数（相同）
}

int main() {
    ShowCount(1, 3.14, "hello", 'c');  // 参数个数: 4
    ShowCount();                        // 参数个数: 0

    return 0;
}
```

### 2.3 递归展开参数包

C++11 中展开参数包的经典方式是**递归**：提供一个通用版本和一个终止版本。

```cpp
#include <iostream>
using namespace std;

// 终止条件：无参数时停止递归
void PrintAll() {
    cout << endl;
}

// 递归版本：取出第一个参数，剩余参数继续递归
template <typename T, typename... Rest>
void PrintAll(T first, Rest... rest) {
    cout << first << " ";
    PrintAll(rest...);  // 递归展开
}

int main() {
    PrintAll(1, 2.5, "three", '4');
    // 输出: 1 2.5 three 4

    return 0;
}
```

递归展开的过程：
1. `PrintAll(1, 2.5, "three", '4')` → 打印 `1`，调用 `PrintAll(2.5, "three", '4')`
2. `PrintAll(2.5, "three", '4')` → 打印 `2.5`，调用 `PrintAll("three", '4')`
3. `PrintAll("three", '4')` → 打印 `three`，调用 `PrintAll('4')`
4. `PrintAll('4')` → 打印 `4`，调用 `PrintAll()`
5. `PrintAll()` → 打印换行，递归结束

### 2.4 逗号表达式展开技巧

利用初始化列表和逗号表达式，可以避免写递归终止函数：

```cpp
#include <iostream>
using namespace std;

template <typename... Args>
void PrintCompact(Args... args) {
    // 利用初始化列表展开参数包
    int dummy[] = { (cout << args << " ", 0)... };
    (void)dummy;  // 消除未使用变量警告
    cout << endl;
}

int main() {
    PrintCompact(10, 20, 30, 40);
    // 输出: 10 20 30 40

    return 0;
}
```

### 2.5 可变参数类模板

类模板同样支持可变参数，最典型的应用是 `std::tuple`：

```cpp
#include <iostream>
using namespace std;

// 简化的 Tuple 实现
template <typename... Types>
class SimpleTuple;

// 终止特化
template <>
class SimpleTuple<> {
public:
    void Print() const { cout << ")" << endl; }
};

// 递归继承展开
template <typename Head, typename... Tail>
class SimpleTuple<Head, Tail...> : public SimpleTuple<Tail...> {
    Head value;
public:
    SimpleTuple(Head h, Tail... t) : value(h), SimpleTuple<Tail...>(t...) {}

    void Print() const {
        cout << value << ", ";
        SimpleTuple<Tail...>::Print();
    }
};

int main() {
    SimpleTuple<int, double, string> t(42, 3.14, "hello");
    cout << "(";
    t.Print();
    // 输出: (42, 3.14, hello, )

    return 0;
}
```

> **C++17 折叠表达式**：C++17 引入了折叠表达式（Fold Expression），可以用 `(cout << ... << args)` 一行完成展开，无需递归。这是后话，此处仅做提及。

---

## 三、Lambda 表达式语法详解

### 3.1 为什么需要 Lambda

在 C++11 之前，如果想给 `std::sort` 传入自定义比较规则，必须定义一个函数对象或函数指针：

```cpp
// C++03 的做法：定义一个结构体
struct CompareByLength {
    bool operator()(const string& a, const string& b) const {
        return a.size() < b.size();
    }
};

sort(words.begin(), words.end(), CompareByLength());
```

这种方式的问题：比较逻辑只在这一处使用，却要单独定义一个类型，代码分散、阅读时需要跳转。

Lambda 让你**就地定义匿名函数**：

```cpp
sort(words.begin(), words.end(),
     [](const string& a, const string& b) {
         return a.size() < b.size();
     });
```

逻辑就在使用的地方，一目了然。

### 3.2 完整语法

Lambda 表达式的完整形式如下：

```
[捕获列表](参数列表) mutable noexcept -> 返回类型 { 函数体 }
```

各部分说明：

| 组成部分 | 是否必须 | 说明 |
|---------|---------|------|
| `[捕获列表]` | 是 | 指定从外部作用域捕获哪些变量 |
| `(参数列表)` | 否（无参时可省略） | 与普通函数参数相同 |
| `mutable` | 否 | 允许修改值捕获的变量（副本） |
| `noexcept` | 否 | 声明不抛出异常 |
| `-> 返回类型` | 否 | 编译器通常能自动推导 |
| `{ 函数体 }` | 是 | 实际执行的代码 |

### 3.3 捕获列表详解

捕获列表决定了 Lambda 如何访问外部变量：

```cpp
#include <iostream>
#include <string>
using namespace std;

int main() {
    int x = 10;
    int y = 20;
    string name = "world";

    // 1. 值捕获（拷贝）：Lambda 内部持有副本
    auto f1 = [x, y]() {
        cout << x + y << endl;  // 30
        // x = 100;  // 编译错误！值捕获默认是 const 的
    };

    // 2. 引用捕获：直接操作外部变量
    auto f2 = [&x, &y]() {
        x = 100;
        y = 200;
    };

    // 3. 隐式值捕获 [=]：所有外部变量按值捕获
    auto f3 = [=]() {
        cout << x << " " << y << " " << name << endl;
    };

    // 4. 隐式引用捕获 [&]：所有外部变量按引用捕获
    auto f4 = [&]() {
        x += 1;
        y += 1;
    };

    // 5. 混合捕获：[=, &x] 表示 x 引用捕获，其余值捕获
    auto f5 = [=, &x]() {
        x = 999;       // x 是引用，可修改
        cout << y << endl;  // y 是值副本
    };

    f1();
    f2();
    cout << x << " " << y << endl;  // 100 200

    return 0;
}
```

### 3.4 mutable 关键字

值捕获的变量在 Lambda 内部默认是 `const` 的。如果需要修改副本（不影响外部），使用 `mutable`：

```cpp
int main() {
    int count = 0;

    // 每次调用，内部 count 副本递增（外部 count 不变）
    auto counter = [count]() mutable {
        ++count;
        cout << "内部 count: " << count << endl;
    };

    counter();  // 内部 count: 1
    counter();  // 内部 count: 2
    counter();  // 内部 count: 3

    cout << "外部 count: " << count << endl;  // 0（未改变）

    return 0;
}
```

### 3.5 返回类型推导与显式指定

大多数情况下，编译器能自动推导 Lambda 的返回类型。但当函数体有多条 return 且类型不同时，需要显式指定：

```cpp
#include <iostream>
using namespace std;

int main() {
    // 自动推导返回类型为 double
    auto divide = [](int a, int b) {
        return static_cast<double>(a) / b;
    };

    // 显式指定返回类型
    auto process = [](int x) -> string {
        if (x > 0) return "positive";
        else return "non-positive";
    };

    cout << divide(7, 2) << endl;     // 3.5
    cout << process(5) << endl;       // positive
    cout << process(-1) << endl;      // non-positive

    return 0;
}
```

---

## 四、闭包类型与 std::function

### 4.1 闭包类型

每个 Lambda 表达式都会生成一个**独一无二的匿名类型**（称为闭包类型），编译器为其自动生成 `operator()`。两个看起来相同的 Lambda，其类型也不同：

```cpp
auto f1 = [](int x) { return x * 2; };
auto f2 = [](int x) { return x * 2; };

// f1 和 f2 类型不同！不能用一个变量同时存储两者
// 但可以用 auto 分别接收
```

### 4.2 std::function：通用函数包装器

当需要存储、传递 Lambda（或其他可调用对象）时，`std::function` 提供了类型擦除的统一接口：

```cpp
#include <iostream>
#include <functional>
#include <vector>
using namespace std;

int main() {
    // 用 std::function 存储不同类型的可调用对象
    function<int(int, int)> op;

    op = [](int a, int b) { return a + b; };
    cout << op(3, 4) << endl;  // 7

    op = [](int a, int b) { return a * b; };
    cout << op(3, 4) << endl;  // 12

    // 存储一组操作
    vector<function<int(int)>> transforms;
    transforms.push_back([](int x) { return x * 2; });
    transforms.push_back([](int x) { return x + 10; });
    transforms.push_back([](int x) { return x * x; });

    for (const auto& f : transforms) {
        cout << f(5) << " ";  // 10 15 25
    }
    cout << endl;

    return 0;
}
```

### 4.3 auto vs std::function

| 对比项 | `auto` | `std::function` |
|--------|--------|-----------------|
| 类型 | 精确的闭包类型 | 类型擦除的包装器 |
| 性能 | 零开销，可内联 | 有虚函数/堆分配开销 |
| 灵活性 | 一个变量只能存一种 Lambda | 可存储任何兼容签名的可调用对象 |
| 使用场景 | 局部使用、模板参数 | 回调存储、容器存放多种函数 |

> **建议**：能用 `auto` 就用 `auto`，只有在需要类型擦除（如存入容器、作为非模板参数传递）时才使用 `std::function`。

---

## 五、Lambda 与 STL 算法实战

### 5.1 排序

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <string>
using namespace std;

struct Student {
    string name;
    int score;
};

int main() {
    vector<Student> students{
        {"Alice", 92}, {"Bob", 85}, {"Carol", 97}, {"Dave", 88}
    };

    // 按分数降序排列
    sort(students.begin(), students.end(),
         [](const Student& a, const Student& b) {
             return a.score > b.score;
         });

    for (const auto& s : students) {
        cout << s.name << ": " << s.score << endl;
    }

    return 0;
}
```

### 5.2 查找与计数

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
using namespace std;

int main() {
    vector<int> nums{3, 1, 4, 1, 5, 9, 2, 6, 5, 3};

    // 统计大于 4 的元素个数
    int count = count_if(nums.begin(), nums.end(),
                         [](int x) { return x > 4; });
    cout << "大于4的个数: " << count << endl;  // 4

    // 查找第一个偶数
    auto it = find_if(nums.begin(), nums.end(),
                      [](int x) { return x % 2 == 0; });
    if (it != nums.end()) {
        cout << "第一个偶数: " << *it << endl;  // 4
    }

    return 0;
}
```

### 5.3 变换与累积

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <numeric>
using namespace std;

int main() {
    vector<int> nums{1, 2, 3, 4, 5};
    vector<int> squares(nums.size());

    // transform：对每个元素求平方
    transform(nums.begin(), nums.end(), squares.begin(),
              [](int x) { return x * x; });

    for (int x : squares) cout << x << " ";  // 1 4 9 16 25
    cout << endl;

    // accumulate：自定义累积操作（求乘积）
    int product = accumulate(nums.begin(), nums.end(), 1,
                             [](int acc, int x) { return acc * x; });
    cout << "乘积: " << product << endl;  // 120

    return 0;
}
```

### 5.4 自定义删除与过滤

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
using namespace std;

int main() {
    vector<int> nums{1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

    // 删除所有奇数（erase-remove 惯用法）
    nums.erase(
        remove_if(nums.begin(), nums.end(),
                  [](int x) { return x % 2 != 0; }),
        nums.end()
    );

    for (int x : nums) cout << x << " ";  // 2 4 6 8 10
    cout << endl;

    return 0;
}
```

---

## 六、Lambda 常见陷阱与最佳实践

### 6.1 悬垂引用

引用捕获的变量如果在 Lambda 执行时已经销毁，就会产生悬垂引用：

```cpp
#include <iostream>
#include <functional>
using namespace std;

function<int()> CreateCounter() {
    int count = 0;
    // 危险！count 是局部变量，函数返回后即销毁
    return [&count]() { return ++count; };  // 悬垂引用！
}

// 正确做法：值捕获 + mutable
function<int()> CreateCounterSafe() {
    int count = 0;
    return [count]() mutable { return ++count; };  // 安全：持有副本
}

int main() {
    auto counter = CreateCounterSafe();
    cout << counter() << endl;  // 1
    cout << counter() << endl;  // 2
    cout << counter() << endl;  // 3

    return 0;
}
```

> **规则**：如果 Lambda 的生命周期超过捕获变量的生命周期，**必须值捕获**。

### 6.2 循环中的引用捕获

```cpp
#include <iostream>
#include <vector>
#include <functional>
using namespace std;

int main() {
    vector<function<void()>> callbacks;

    for (int i = 0; i < 5; ++i) {
        // 错误：所有 Lambda 共享同一个 i 的引用
        // callbacks.push_back([&]() { cout << i << " "; });

        // 正确：值捕获当前 i 的快照
        callbacks.push_back([i]() { cout << i << " "; });
    }

    for (const auto& cb : callbacks) cb();
    // 正确输出: 0 1 2 3 4
    cout << endl;

    return 0;
}
```

### 6.3 捕获 this 指针

在成员函数中使用 Lambda 时，常需要捕获 `this`：

```cpp
#include <iostream>
#include <algorithm>
#include <vector>
using namespace std;

class Team {
    vector<int> scores;
    int bonus;

public:
    Team(vector<int> s, int b) : scores(s), bonus(b) {}

    void ApplyBonus() {
        // 捕获 this，访问成员变量
        for_each(scores.begin(), scores.end(),
                 [this](int& score) { score += bonus; });
    }

    void Print() const {
        for (int s : scores) cout << s << " ";
        cout << endl;
    }
};

int main() {
    Team t({80, 85, 90}, 5);
    t.ApplyBonus();
    t.Print();  // 85 90 95

    return 0;
}
```

> C++20 中 `[=]` 不再隐式捕获 `this`，需显式写 `[=, this]` 或 `[=, *this]`（捕获对象副本）。

### 6.4 最佳实践总结

1. **优先值捕获**：除非有明确的性能理由或需要修改外部变量。
2. **警惕生命周期**：Lambda 存储/异步执行时，确保捕获的变量仍然存活。
3. **避免 `[=]` 和 `[&]` 的隐式捕获**：显式列出需要的变量，意图更清晰。
4. **短小精悍**：Lambda 体超过 10 行时，考虑提取为命名函数。
5. **用 `auto` 接收**：避免不必要的 `std::function` 开销。

---

## 七、总结

### 核心要点回顾

| 知识点 | 关键要点 |
|--------|---------|
| `>>` 修正 | 嵌套模板不再需要空格 |
| 模板别名 `using` | 替代 typedef，支持模板级别名 |
| `extern template` | 避免重复实例化，加速编译 |
| 可变参数模板 | 参数包 + 递归展开，实现任意参数泛型 |
| Lambda 语法 | `[捕获](参数) mutable -> 返回类型 { 体 }` |
| 闭包与 `std::function` | 闭包是精确类型，`std::function` 是类型擦除包装 |
| Lambda 与 STL | 就地定义比较、谓词、变换逻辑 |

### 模板与 Lambda 的协同

模板提供**编译期多态**，Lambda 提供**轻量级函数对象**。两者结合，让 C++ 泛型代码既高效又易读：

```cpp
// 通用函数：接受任何可调用对象
template <typename Func>
void ApplyTwice(Func f, int x) {
    f(x);
    f(x);
}

// 调用时传入 Lambda，无需定义额外的类
ApplyTwice([](int v) { cout << v << endl; }, 42);
```

下一篇，我们将深入 C++11 智能指针的世界，看看 RAII 思想如何彻底解放手动内存管理的负担。
