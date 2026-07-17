# 08 C++STL概述与序列式容器

## 一、STL 的前世今生

### 1.1 从重复劳动到标准库

在 C++ 诞生后的很长一段时间里，程序员面临的处境颇为尴尬：语言本身提供了一套强大的面向对象和泛型编程机制，但日常开发中反复用到的数据结构与算法——数组、链表、排序、查找——却始终没有统一的标准实现。每个团队、每个项目都在"重新发明轮子"，代码风格参差不齐，质量良莠不齐，维护成本居高不下。

这种局面在 1994 年发生了根本性的改变。当时在惠普实验室工作的 Alexander Stepanov 和 Meng Lee 提出了一个大胆的方案：利用 C++ 的**模板机制**，将数据结构与算法以**正交分解**的方式设计——容器只管存储数据，算法只管处理逻辑，两者通过一个统一的"中间人"（迭代器）衔接。这套方案后来被正式纳入 C++ 标准，即 **STL（Standard Template Library，标准模板库）**。

STL 的设计哲学可以概括为一句话：**数据结构与算法的彻底分离**。这听起来抽象，但理解它之后，你就能明白为什么 `sort()` 既能给 `vector` 排序也能给 `deque` 排序——因为 `sort` 不关心数据在内存中如何排布，它只通过迭代器来"看到"数据。

### 1.2 STL 的六大组件

STL 的内部结构由六大组件构成，每一组件各司其职：

| 组件 | 中文名称 | 职责 | 具体体现 |
|------|---------|------|---------|
| Container | 容器 | 存储数据 | `vector`, `list`, `deque`, `set`, `map` 等 |
| Algorithm | 算法 | 处理数据 | `sort`, `find`, `for_each`, `copy` 等 |
| Iterator | 迭代器 | 连接容器与算法 | 每种容器都有专属的迭代器类型 |
| Functor | 仿函数 | 为算法提供策略 | `greater<int>()`, 自定义排序规则等 |
| Adapter | 适配器 | 修饰容器或仿函数接口 | `stack`, `queue` 就是对 `deque` 的适配 |
| Allocator | 空间配置器 | 管理内存分配 | `std::allocator`，通常使用默认即可 |

这六大组件的关系可以这样理解：**容器**是仓库，**算法**是工人，**迭代器**是工人进入仓库取货的通道，**仿函数**是下达给工人的具体指令，**适配器**是对仓库入口的改造，**空间配置器**则负责仓库内部货架的搭建与拆除。

---

## 二、迭代器：容器与算法的"粘合剂"

### 2.1 迭代器的本质

初学阶段，把迭代器理解为**一种特殊的指针**是最快上手的方式。像指针一样，迭代器可以通过 `*` 解引用访问元素，通过 `++` 移动到下一个位置。但它比指针更"聪明"——不同容器提供不同能力的迭代器，这种能力差异是 STL 精巧设计的一部分。

```cpp
vector<int> v;
v.push_back(10);
v.push_back(20);

// 迭代器的使用方式与指针高度相似
vector<int>::iterator it = v.begin();  // 指向第一个元素
cout << *it << endl;                    // 输出 10
it++;                                   // 移动到下一个
cout << *it << endl;                    // 输出 20
```

### 2.2 迭代器的五种类型

并非所有迭代器都生而平等。STL 定义了五种迭代器类型，能力逐级递增：

| 迭代器类型 | 能力 | 支持操作 | 典型容器 |
|-----------|------|---------|---------|
| 输入迭代器 | 只读，单次单向 | `++`, `==`, `!=`, `*`（读） | `istream_iterator` |
| 输出迭代器 | 只写，单次单向 | `++`, `*`（写） | `ostream_iterator` |
| 前向迭代器 | 读写，可多次单向 | 输入+输出迭代器的能力 | `forward_list` 的迭代器 |
| 双向迭代器 | 读写，可前后移动 | 前向迭代器能力 + `--` | `list`, `set`, `map` 的迭代器 |
| 随机访问迭代器 | 读写，任意跳跃 | 双向迭代器能力 + `[]`, `+n`, `-n`, `<`, `>` | `vector`, `deque`, `string` 的迭代器 |

这个分类解释了为什么 `sort()` 能用在 `vector` 上却不能用在 `list` 上——`sort` 内部需要随机跳跃访问元素，而 `list` 只提供了双向迭代器，不支持 `it + 5` 这样的操作。作为补偿，`list` 提供了自己的 `sort()` 成员函数。

---

## 三、容器总览：序列式 vs 关联式

STL 的容器家族分为两大分支：

**序列式容器**：元素的位置由插入时机决定，每个元素有固定的物理位置：
- `string` —— 字符序列
- `vector` —— 单端动态数组
- `deque` —— 双端动态数组
- `list` —— 双向链表
- `stack` / `queue` —— 适配器容器

**关联式容器**：元素的位置由排序规则决定，插入时自动确定位置（将在下一篇文章详解）：
- `set` / `multiset` —— 集合
- `map` / `multimap` —— 映射表

本文聚焦于序列式容器，下面逐一深入讲解。

---

## 四、string 容器：不只是字符串

### 4.1 string 的本质

很多 C++ 初学者会把 `string` 和 `char*` 混为一谈，这是危险的误解。`char*` 是一个裸指针，它指向一块字符数组，但你不知道这块内存有多大、能否安全写入；`string` 是一个**类**，它在内部封装了 `char*` 数组，并提供了完整的内存管理、边界检查和丰富的操作接口。

```cpp
// char* 的风险
char* s1 = "hello";        // s1 指向只读的字符串字面量
// s1[0] = 'H';            // 未定义行为！

// string 的安全
string s2 = "hello";
s2[0] = 'H';               // 完全安全，s2 内部维护了可写的缓冲区
```

### 4.2 构造与初始化

`string` 提供了多种构造函数，覆盖了常见的使用场景：

```cpp
#include <iostream>
#include <string>
using namespace std;

int main() {
    string s1;                          // 默认构造：空字符串
    cout << "s1: [" << s1 << "]" << endl;

    string s2("hello world");           // C 风格字符串构造
    cout << "s2: " << s2 << endl;

    string s3(s2);                      // 拷贝构造
    cout << "s3: " << s3 << endl;

    string s4(5, '*');                  // 5 个字符构造
    cout << "s4: " << s4 << endl;

    string s5(s2, 0, 5);               // 从 s2 的第 0 位取 5 个字符："hello"
    cout << "s5: " << s5 << endl;

    return 0;
}
```

### 4.3 赋值操作

`string` 重载了 `=` 运算符，同时提供了 `assign` 方法族，比 `char*` 的 `strcpy` 安全得多：

```cpp
void TestAssign() {
    string str;
    
    str = "hello";                       // 直接赋值 C 字符串
    cout << str << endl;                 // hello

    str = 'X';                          // 赋单个字符
    cout << str << endl;                 // X

    str.assign("welcome to C++");       // assign 方式
    cout << str << endl;                 // welcome to C++

    str.assign("welcome to C++", 7);    // 只取前 7 个字符
    cout << str << endl;                 // welcome

    str.assign(5, '-');                 // 5 个相同字符
    cout << str << endl;                 // -----
}
```

### 4.4 字符串拼接

`string` 的拼接操作是它相比 `char*` 最直观的优势之一——你不再需要手动计算缓冲区大小然后调用 `strcat`：

```cpp
void TestAppend() {
    string s = "C++";
    
    s += " is";                          // 拼接 C 字符串
    s += ' ';                           // 拼接单个字符
    s += "powerful";                    // 再次拼接
    cout << s << endl;                  // C++ is powerful

    string s2 = " Programming";
    s.append(s2);                       // 拼接整个 string
    cout << s << endl;                  // C++ is powerful Programming

    s.append(s2, 0, 4);                // 从 s2 第 0 位取 4 个字符：" Pro"
    cout << s << endl;                  // ... Programming Pro

    string s3 = "Language";
    s.append(s3, 4, 4);                // 从 s3 第 4 位取 4 个:"uage"
    cout << s << endl;
}
```

> **拼接的内部机制**：每次拼接时 `string` 会自动检测内部缓冲区是否足够，不够则自动扩容，程序员完全不需要关心内存管理。这正是类封装带来的好处。

### 4.5 查找与替换

`find` 和 `rfind` 是 `string` 中使用频率极高的方法：

```cpp
void TestFind() {
    string s = "hello world, hello C++";

    int pos = s.find("hello");          // 从左查找，找到返回位置
    if (pos != string::npos) {          // string::npos 即为 -1
        cout << "第一次出现位置: " << pos << endl;  // 0
    }

    pos = s.rfind("hello");             // 从右查找
    cout << "最后一次出现位置: " << pos << endl;    // 13

    pos = s.find("java");               // 找不到返回 string::npos
    if (pos == string::npos) {
        cout << "未找到" << endl;
    }

    // 替换：从位置 0 开始，替换 5 个字符为 "HELLO"
    s.replace(0, 5, "HELLO");
    cout << s << endl;                  // HELLO world, hello C++
}
```

> **注意**：`string::npos` 是 `find` 方法的"哨兵值"，定义为 `-1`，但由于它是 `string::size_type` 类型（无符号整数），实际表现为一个极大的正数。判断时永远用 `string::npos` 而不要用 `-1`。

### 4.6 字符存取与遍历

`string` 支持两种随机访问方式——`operator[]` 和 `at()`：

```cpp
void TestAccess() {
    string s = "hello";
    
    // [] 方式：不检查越界，性能更高
    for (size_t i = 0; i < s.size(); i++) {
        cout << s[i] << " ";
    }
    cout << endl;

    // at() 方式：检查越界，越界时抛出 out_of_range 异常
    for (size_t i = 0; i < s.size(); i++) {
        cout << s.at(i) << " ";
    }
    cout << endl;

    // 修改字符
    s[0] = 'H';
    s.at(4) = 'O';
    cout << s << endl;                  // HellO
}
```

> **选择建议**：已经确认索引不会越界时，用 `[]` 获得更好性能；处理用户输入或其他不确定索引时，用 `at()` 获得安全的异常提示。

### 4.7 插入、删除与子串

```cpp
void TestInsertErase() {
    string s = "hello";
    
    s.insert(1, "xx");                  // 在位置 1 插入
    cout << s << endl;                  // hxxello

    s.erase(1, 2);                      // 从位置 1 删除 2 个字符
    cout << s << endl;                  // hello

    // 子串：从位置 2 取 3 个字符
    string sub = s.substr(2, 3);
    cout << "sub: " << sub << endl;     // llo

    // 实用场景：解析邮箱
    string email = "zhangsan@gmail.com";
    int atPos = email.find('@');
    string username = email.substr(0, atPos);
    string domain = email.substr(atPos + 1);
    cout << "用户名: " << username << endl;  // zhangsan
    cout << "域名: " << domain << endl;      // gmail.com
}
```

---

## 五、vector 容器：最常用的动态数组

### 5.1 vector 的核心特性

`vector` 是 STL 中使用频率最高的容器，没有之一。它的行为与数组高度相似，但最关键的区别在于：**数组的长度是编译期固定的，而 vector 可以在运行时动态扩展**。

```cpp
// 数组的局限
int arr[10];                // 长度固定为 10，无法改变

// vector 的灵活
vector<int> v;              // 初始为空
v.push_back(1);             // 动态增长
v.push_back(2);
v.push_back(3);             // 现在有 3 个元素
```

**动态扩展的内部原理**：当 `vector` 空间不足时，它的处理方式不是"在原空间后面接一段"，而是：
1. 申请一块更大的新内存
2. 将原数据逐个拷贝到新空间
3. 释放原内存

这意味着每次扩容都有一定的性能开销。因此，如果预先知道数据量，使用 `reserve()` 提前分配好空间是一个很好的习惯。

### 5.2 构造与赋值

```cpp
#include <iostream>
#include <vector>
using namespace std;

void PrintVector(const vector<int>& v) {
    for (vector<int>::const_iterator it = v.begin(); it != v.end(); ++it) {
        cout << *it << " ";
    }
    cout << endl;
}

void TestConstruct() {
    vector<int> v1;                          // 默认构造
    for (int i = 0; i < 5; i++) {
        v1.push_back(i * 10);               // 10, 20, 30, 40, 50
    }
    PrintVector(v1);

    vector<int> v2(v1.begin(), v1.end());   // 区间构造
    PrintVector(v2);

    vector<int> v3(5, 100);                 // 5 个 100
    PrintVector(v3);

    vector<int> v4(v3);                     // 拷贝构造
    PrintVector(v4);
}
```

### 5.3 容量与大小

这是 `vector` 初学者最容易混淆的两个概念：

| 概念 | 英文 | 含义 | 获取方法 |
|------|------|------|---------|
| 大小 | size | 容器中实际存储的元素个数 | `size()` |
| 容量 | capacity | 容器当前分配的内存最多能容纳的元素数 | `capacity()` |

```cpp
void TestCapacity() {
    vector<int> v;
    
    for (int i = 0; i < 100; i++) {
        v.push_back(i);
        // 容量通常按 1.5 倍或 2 倍的倍数增长
        if (i < 10) {
            cout << "size=" << v.size() 
                 << " capacity=" << v.capacity() << endl;
        }
    }

    cout << "最终 —— size=" << v.size() 
         << " capacity=" << v.capacity() << endl;

    // resize 可以强制改变 size
    v.resize(10);            // 截断到 10 个元素，capacity 不变
    cout << "resize(10) —— size=" << v.size() 
         << " capacity=" << v.capacity() << endl;
}
```

### 5.4 插入与删除

```cpp
void TestInsertErase() {
    vector<int> v;
    
    // 尾插——最常用的 vector 操作
    v.push_back(10);
    v.push_back(20);
    v.push_back(30);
    v.push_back(40);
    v.push_back(50);
    PrintVector(v);                        // 10 20 30 40 50

    v.pop_back();                          // 尾删
    PrintVector(v);                        // 10 20 30 40

    // 在指定迭代器位置插入
    v.insert(v.begin(), 100);             // 头部插入 100
    PrintVector(v);                        // 100 10 20 30 40

    v.insert(v.begin() + 2, 3, 200);      // 在位置 2 插入 3 个 200
    PrintVector(v);                        // 100 10 200 200 200 20 30 40

    // 删除指定迭代器位置
    v.erase(v.begin());                   // 删除第一个元素
    PrintVector(v);

    v.erase(v.begin(), v.begin() + 3);    // 删除区间 [begin, begin+3)
    PrintVector(v);

    v.clear();                            // 清空
    cout << "清空后 size=" << v.size() << endl;  // 0
}
```

> **性能提示**：在 `vector` 头部或中间插入/删除元素时，需要移动后续所有元素，时间复杂度为 O(n)。如果需要频繁在头部操作，应该使用 `deque` 或 `list`。

### 5.5 数据存取

```cpp
void TestAccess() {
    vector<int> v;
    for (int i = 0; i < 5; i++) {
        v.push_back(i);
    }

    // 三种访问方式
    cout << v[2] << endl;      // [] 运算符
    cout << v.at(2) << endl;   // at() 方法（带越界检查）
    
    cout << v.front() << endl; // 第一个元素：0
    cout << v.back() << endl;  // 最后一个元素：4
}
```

### 5.6 swap 的妙用：收缩内存

`vector` 有一个经典技巧——利用 `swap` 收缩过大的容量：

```cpp
void TestSwapShrink() {
    vector<int> v;
    for (int i = 0; i < 10000; i++) {
        v.push_back(i);
    }
    cout << "size=" << v.size() 
         << " capacity=" << v.capacity() << endl;
    // size=10000 capacity=16384（具体值取决于实现）

    v.resize(10);                         // 只保留了 10 个元素
    cout << "resize后 size=" << v.size() 
         << " capacity=" << v.capacity() << endl;
    // size=10 capacity=16384 —— 容量还是很大！

    // 收缩技巧：用当前内容构造临时 vector，然后 swap
    vector<int>(v).swap(v);
    cout << "swap后 size=" << v.size() 
         << " capacity=" << v.capacity() << endl;
    // size=10 capacity=10
}
```

这里 `vector<int>(v)` 创建了一个匿名临时对象，它只拷贝了 `v` 中实际存在的 10 个元素，容量恰好等于 10。然后 `swap` 交换了两个对象的内存指针，原来的大块内存随匿名对象一起被销毁。

### 5.7 reserve：预留空间

如果你能预先估计数据量，`reserve` 可以避免反复扩容的开销：

```cpp
void TestReserve() {
    vector<int> v;
    v.reserve(10000);                     // 提前申请 10000 个元素的空间

    int* pFirst = &v[0];
    int reallocCount = 0;

    for (int i = 0; i < 10000; i++) {
        v.push_back(i);
        if (pFirst != &v[0]) {            // 首地址变了说明发生了扩容
            pFirst = &v[0];
            reallocCount++;
        }
    }
    cout << "扩容次数: " << reallocCount << endl;  // 0
}
```

---

## 六、deque 容器：双端开口的利器

### 6.1 deque 与 vector 的本质区别

`deque`（double-ended queue，双端队列）在很多方面与 `vector` 相似——都支持随机访问、都提供 `[]` 运算符。但它们的内存模型截然不同：

- **vector**：一整块连续内存，像一条直线
- **deque**：分段的连续内存，由一个中控器（指针数组）管理多个固定大小的缓冲区段，像一节节车厢连成的火车

这种结构差异导致了性能差异：

| 操作 | vector | deque |
|------|--------|-------|
| 尾部插入/删除 | 快 | 快 |
| 头部插入/删除 | 慢（需移动所有元素） | 快 |
| 随机访问 | 最快（一次寻址） | 稍慢（两次寻址） |
| 中间插入/删除 | 慢 | 慢 |

### 6.2 deque 的独特操作

```cpp
#include <iostream>
#include <deque>
#include <algorithm>
using namespace std;

void PrintDeque(const deque<int>& d) {
    for (deque<int>::const_iterator it = d.begin(); it != d.end(); ++it) {
        cout << *it << " ";
    }
    cout << endl;
}

void TestDeque() {
    deque<int> d;
    
    // 尾部操作（与 vector 一样）
    d.push_back(10);
    d.push_back(20);
    d.push_back(30);
    
    // 头部操作（vector 不具备的能力）
    d.push_front(5);
    d.push_front(1);
    
    PrintDeque(d);                         // 1 5 10 20 30

    d.pop_back();                          // 尾删
    d.pop_front();                         // 头删
    PrintDeque(d);                         // 5 10 20
}
```

### 6.3 deque 的排序

需要注意的是，`deque` 没有 `capacity()` 方法——因为它分段的内部结构，容量这个概念对它没有意义。另外 `deque` 支持用标准算法 `sort` 排序：

```cpp
void TestDequeSort() {
    deque<int> d;
    d.push_back(30);
    d.push_back(10);
    d.push_back(50);
    d.push_back(20);
    d.push_back(40);

    cout << "排序前: ";
    PrintDeque(d);                         // 30 10 50 20 40

    sort(d.begin(), d.end());             // 全局 sort 函数
    cout << "排序后: ";
    PrintDeque(d);                         // 10 20 30 40 50
}
```

---

## 七、stack 容器：先进后出的"弹夹"

### 7.1 什么是适配器容器

`stack` 和 `queue` 不是"原生"容器——它们本身不直接存储数据，而是在已有容器（默认为 `deque`）的基础上，限制访问方式，提供特定的接口。这种设计模式称为**适配器（Adapter）**模式。

`stack` 模拟的是一种"先进后出"（FILO）的结构。可以把 `stack` 想象成一个子弹夹：最先压进去的子弹最后才能打出来，只有顶端的子弹能被触碰。

### 7.2 stack 的接口

`stack` 的接口非常精简，而且**不提供迭代器**——这很合理，因为栈只允许访问栈顶，不支持遍历：

```cpp
#include <iostream>
#include <stack>
using namespace std;

void TestStack() {
    stack<int> s;
    
    s.push(10);                            // 压栈
    s.push(20);
    s.push(30);

    cout << "当前栈大小: " << s.size() << endl;   // 3

    while (!s.empty()) {
        cout << "栈顶: " << s.top() << endl;      // 查看栈顶
        s.pop();                                  // 弹栈
    }

    cout << "弹出后大小: " << s.size() << endl;   // 0
}
```

输出：
```
当前栈大小: 3
栈顶: 30
栈顶: 20
栈顶: 10
弹出后大小: 0
```

注意顺序：30 是最**后**压栈的，却最**先**弹出——这就是"先进后出"。

---

## 八、queue 容器：先进先出的"排队"

### 8.1 queue 的语义

`queue` 模拟的是日常生活中的排队——先来的先服务。在数据结构术语中称为 FIFO（First In First Out）。队列有两个出口：一个入口（队尾），一个出口（队头）。

与 `stack` 一样，`queue` 也是适配器容器，默认底层实现也是 `deque`，同样**不提供迭代器**。

### 8.2 queue 的接口

```cpp
#include <iostream>
#include <queue>
#include <string>
using namespace std;

class Person {
public:
    string name;
    int age;
    Person(string n, int a) : name(n), age(a) {}
};

void TestQueue() {
    queue<Person> q;
    
    q.push(Person("张三", 25));           // 入队
    q.push(Person("李四", 30));
    q.push(Person("王五", 28));

    while (!q.empty()) {
        Person& frontPerson = q.front();  // 队头
        Person& backPerson = q.back();    // 队尾
        cout << "队头: " << frontPerson.name 
             << " | 队尾: " << backPerson.name << endl;
        q.pop();                          // 出队
    }

    cout << "队列大小: " << q.size() << endl;   // 0
}
```

输出：
```
队头: 张三 | 队尾: 王五
队头: 李四 | 队尾: 王五
队头: 王五 | 队尾: 王五
队列大小: 0
```

注意：张三最先入队，也最先出队——"先进先出"。而队尾始终是最后入队的那个人（王五），直到他也出队为止。

---

## 九、list 容器：灵活的双向链表

### 9.1 list 的物理结构

`list` 是一个**双向链表**。每个元素存储在一个独立的内存节点（Node）中，节点除了保存数据，还保存指向前一个节点和后一个节点的指针。

这种结构赋予了 `list` 两个独特优势：
- **插入和删除极快**：只修改相邻节点的指针，不移动任何数据
- **迭代器极稳定**：插入或删除元素后，指向其他元素的迭代器不会失效（`vector` 在扩容时所有迭代器都会失效）

代价是：
- **无法随机访问**：访问第 100 个元素需要从第 1 个开始数 100 步
- **内存开销大**：每个节点额外存储两个指针

### 9.2 list 的构造与基本操作

```cpp
#include <iostream>
#include <list>
using namespace std;

void PrintList(const list<int>& lst) {
    for (list<int>::const_iterator it = lst.begin(); it != lst.end(); ++it) {
        cout << *it << " ";
    }
    cout << endl;
}

void TestList() {
    list<int> lst;
    
    // 尾插和头插
    lst.push_back(30);
    lst.push_back(40);
    lst.push_front(20);
    lst.push_front(10);
    
    PrintList(lst);                        // 10 20 30 40

    // 删除
    lst.pop_back();                        // 尾删：移除 40
    lst.pop_front();                       // 头删：移除 10
    PrintList(lst);                        // 20 30
}
```

### 9.3 list 特有的操作

`list` 有一些其他序列容器不具备的成员函数：

```cpp
void TestListSpecial() {
    list<int> lst;
    lst.push_back(1);
    lst.push_back(2);
    lst.push_back(2);
    lst.push_back(3);
    lst.push_back(2);
    lst.push_back(4);

    PrintList(lst);                        // 1 2 2 3 2 4

    // remove：按值删除所有匹配元素
    lst.remove(2);
    PrintList(lst);                        // 1 3 4

    // reverse：反转链表
    lst.reverse();
    PrintList(lst);                        // 4 3 1
}
```

这两个函数都是 `list` 的专属成员函数——`remove(值)` 不是全局算法 `std::remove`，全局版本需要通过 `erase` 配合才真正删除元素，而 `list::remove` 一步到位。

### 9.4 list 的排序

由于 `list` 的迭代器是**双向迭代器**（不是随机访问迭代器），全局函数 `std::sort` 无法用于 `list`——`std::sort` 要求随机访问迭代器。因此 `list` 不得不提供自己的 `sort()` 成员函数：

```cpp
bool MyCompare(int v1, int v2) {
    return v1 > v2;                        // 降序
}

void TestListSort() {
    list<int> lst;
    lst.push_back(30);
    lst.push_back(10);
    lst.push_back(50);
    lst.push_back(20);
    lst.push_back(40);

    cout << "排序前: ";
    PrintList(lst);                        // 30 10 50 20 40

    lst.sort();                            // 默认升序
    cout << "升序: ";
    PrintList(lst);                        // 10 20 30 40 50

    lst.sort(MyCompare);                   // 自定义降序
    cout << "降序: ";
    PrintList(lst);                        // 50 40 30 20 10
}
```

### 9.5 自定义类型的排序

这是一个经典面试题场景：将一个 `Person` 对象的 `list` 按年龄排序，年龄相同则按身高排序：

```cpp
#include <string>

class Person {
public:
    string m_Name;
    int m_Age;
    int m_Height;

    Person(string name, int age, int height) 
        : m_Name(name), m_Age(age), m_Height(height) {}
};

// 排序规则：按年龄升序，年龄相同按身高降序
bool ComparePerson(const Person& p1, const Person& p2) {
    if (p1.m_Age == p2.m_Age) {
        return p1.m_Height > p2.m_Height;  // 年龄相同时，高的在前
    }
    return p1.m_Age < p2.m_Age;            // 年龄小的在前
}

void TestListPersonSort() {
    list<Person> persons;

    persons.push_back(Person("刘备", 35, 175));
    persons.push_back(Person("曹操", 45, 180));
    persons.push_back(Person("孙权", 40, 170));
    persons.push_back(Person("赵云", 25, 190));
    persons.push_back(Person("张飞", 35, 160));
    persons.push_back(Person("关羽", 35, 200));

    persons.sort(ComparePerson);

    for (list<Person>::iterator it = persons.begin(); 
         it != persons.end(); ++it) {
        cout << "姓名: " << it->m_Name 
             << "  年龄: " << it->m_Age 
             << "  身高: " << it->m_Height << endl;
    }
}
```

输出：
```
姓名: 赵云  年龄: 25  身高: 190
姓名: 关羽  年龄: 35  身高: 200
姓名: 刘备  年龄: 35  身高: 175
姓名: 张飞  年龄: 35  身高: 160
姓名: 孙权  年龄: 40  身高: 170
姓名: 曹操  年龄: 45  身高: 180
```

可以看到，三个 35 岁的人中，关羽 200cm 排最前，张飞 160cm 排最后——二级排序规则完美执行。

---

## 十、序列式容器选型指南

这么多容器，实际开发中该怎么选？下面是一个简明的选择决策表：

| 需求场景 | 推荐容器 | 理由 |
|---------|---------|------|
| 需要随机访问，尾部增删 | `vector` | 最快的随机访问速度 |
| 需要随机访问，头尾频繁增删 | `deque` | 两端操作都很快 |
| 频繁在任意位置插入删除 | `list` | 插入删除 O(1)，迭代器不失效 |
| 需要后进先出 | `stack` | 接口语义明确 |
| 需要先进先出 | `queue` | 接口语义明确 |
| 仅存储字符序列 | `string` | 为字符串特化，接口最方便 |

**黄金法则**：默认就用 `vector`，遇到性能瓶颈再根据具体访问模式切换到其他容器。大多数场景下，`vector` 的连续内存布局带来的缓存友好性足以抵消它在中间插入时的劣势。

---

## 十一、小结

本文从 STL 的历史背景讲起，介绍了 STL 六大组件的职责分工，然后深入讲解了五种迭代器类型的能力层级，最后逐一剖析了六大序列式容器（string、vector、deque、stack、queue、list）的核心特性和使用方式。

掌握了序列式容器，就掌握了 STL 的半壁江山。下一篇文章将转向 STL 的另一大板块——**关联式容器**（set/map）和**函数对象**（仿函数、谓词、内建仿函数），它们将带你进入自动排序和键值查找的世界。
