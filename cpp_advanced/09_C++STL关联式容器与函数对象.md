# 09 C++STL关联式容器与函数对象

## 一、关联式容器：自动排序的奥秘

### 1.1 序列式 vs 关联式：两种不同的世界观

在上一篇文章中，我们接触的都是**序列式容器**——元素的排列顺序由你插入的时机决定。你先 push 谁，谁就排在前面，一切都是程序员可控的。

**关联式容器**走的是一条完全不同的路。它们的底层是一棵**平衡二叉搜索树**（通常是红黑树），元素的"位置"不由插入时机决定，而由元素自身的值（或键）和排序规则共同决定。你插入 `{10, 30, 20}`，它自动帮你排成 `{10, 20, 30}`。

这种"自动化"带来的好处是：
- **查找极快**：二叉树结构使得查找时间复杂度为 O(log n)，而 `vector` 的线性查找是 O(n)
- **始终有序**：任何时候遍历，元素都是按规则排好的，不需要手动 sort
- **插入即排序**：插入一个新元素时，它自动找到该去的位置

代价则是：
- **插入稍慢**：每次插入都需要在树中找到正确位置并调整平衡
- **无法随机访问**：不能像 `vector[5]` 那样通过下标直接取元素
- **迭代器是双向的**：只支持 `++` 和 `--`，不支持 `+n`

### 1.2 set 和 map 家族

关联式容器有四个核心成员：

| 容器 | 键值模型 | 是否允许重复 | 通俗理解 |
|------|---------|-------------|---------|
| `set` | 只有键，值即键 | 不允许 | 不重复的自动排序集合 |
| `multiset` | 只有键，值即键 | 允许 | 可重复的自动排序集合 |
| `map` | 键值对（key→value） | 键不允许重复 | 自动排序的字典 |
| `multimap` | 键值对（key→value） | 键允许重复 | 一个键可对应多个值的字典 |

`set` 和 `map` 是一对"双胞胎中的兄妹"——`set` 中每个元素就是它自己（值即键），`map` 中每个元素由键（key）和值（value）两部分组成，通过键来索引值。

---

## 二、set 容器：不重复的自动排序集合

### 2.1 set 的本质

`set` 就像一个永远不会出现重复元素的、自动保持升序的集合。当你往里面扔三个数字 5、3、7，它会自动整理成 3、5、7；当你试图再扔一个 5 进去，它会拒绝——因为已经有了。

这个特性使得 `set` 特别适合去重和自动排序的场景。

### 2.2 构造与赋值

```cpp
#include <iostream>
#include <set>
using namespace std;

void PrintSet(const set<int>& s) {
    for (set<int>::const_iterator it = s.begin(); it != s.end(); ++it) {
        cout << *it << " ";
    }
    cout << endl;
}

void TestConstruct() {
    set<int> s1;                           // 默认构造
    s1.insert(30);
    s1.insert(10);
    s1.insert(50);
    s1.insert(20);
    s1.insert(40);
    PrintSet(s1);                          // 10 20 30 40 50 —— 自动排序

    set<int> s2(s1);                       // 拷贝构造
    PrintSet(s2);                          // 10 20 30 40 50

    set<int> s3;
    s3 = s1;                               // 赋值
    PrintSet(s3);
}
```

注意：即使插入顺序是 30→10→50→20→40，输出始终是 10→20→30→40→50。**插入即排序**，这是 `set` 最核心的特征。

### 2.3 大小与交换

```cpp
void TestSizeSwap() {
    set<int> s1;
    s1.insert(10);
    s1.insert(20);
    s1.insert(30);

    if (!s1.empty()) {
        cout << "s1 元素个数: " << s1.size() << endl;   // 3
    }

    set<int> s2;
    s2.insert(100);
    s2.insert(200);

    cout << "交换前——" << endl;
    cout << "s1: "; PrintSet(s1);          // 10 20 30
    cout << "s2: "; PrintSet(s2);          // 100 200

    s1.swap(s2);

    cout << "交换后——" << endl;
    cout << "s1: "; PrintSet(s1);          // 100 200
    cout << "s2: "; PrintSet(s2);          // 10 20 30
}
```

> **注意**：`set` 没有 `resize` 方法，也没有 `capacity` 概念——这些只属于序列式容器。`set` 是树结构，每个节点独立分配内存。

### 2.4 插入与删除

```cpp
void TestInsertErase() {
    set<int> s;
    
    // insert 插入
    s.insert(10);
    s.insert(30);
    s.insert(20);
    s.insert(40);
    PrintSet(s);                           // 10 20 30 40

    // erase 的三种形式
    s.erase(s.begin());                    // 按迭代器删除
    PrintSet(s);                           // 20 30 40

    s.erase(30);                           // 按值删除
    PrintSet(s);                           // 20 40

    s.clear();                             // 清空
    cout << "清空后 size=" << s.size() << endl;  // 0
}
```

### 2.5 查找与统计

`set` 的 `find` 比全局 `std::find` 高效得多——前者利用树结构以 O(log n) 完成，后者只能 O(n) 线性扫描：

```cpp
void TestFindCount() {
    set<int> s;
    s.insert(10);
    s.insert(30);
    s.insert(20);
    s.insert(40);

    // find：找到返回迭代器，找不到返回 end()
    set<int>::iterator it = s.find(30);
    if (it != s.end()) {
        cout << "找到了: " << *it << endl; // 30
    }

    it = s.find(50);
    if (it == s.end()) {
        cout << "50 不存在" << endl;
    }

    // count：对于 set，结果只能是 0 或 1
    cout << "30 出现次数: " << s.count(30) << endl;  // 1
    cout << "50 出现次数: " << s.count(50) << endl;  // 0
}
```

### 2.6 set 与 multiset 的区别

这是很多人面试中踩过的坑。`set` 不允许重复元素，`multiset` 允许：

```cpp
#include <set>

void TestSetVsMultiset() {
    // set：不允许重复
    set<int> s;
    pair<set<int>::iterator, bool> ret = s.insert(10);
    cout << "第一次插入 10: " 
         << (ret.second ? "成功" : "失败") << endl;  // 成功

    ret = s.insert(10);
    cout << "第二次插入 10: " 
         << (ret.second ? "成功" : "失败") << endl;  // 失败！

    // multiset：允许重复
    multiset<int> ms;
    ms.insert(10);
    ms.insert(10);
    ms.insert(10);
    cout << "multiset 中 10 的个数: " 
         << ms.count(10) << endl;                    // 3
}
```

关键是 `set::insert` 的返回值——它返回一个 `pair<iterator, bool>`：
- `first`：指向插入元素的迭代器（插入成功时）或已存在元素的迭代器（插入失败时）
- `second`：`true` 表示插入成功，`false` 表示已有相同元素

### 2.7 pair 对组

`pair` 是一个简单的二元组模板，能将两个值"打包"成一个值来传递。在 `map` 和 `set` 的接口中无处不在：

```cpp
#include <string>

void TestPair() {
    // 方式一：构造函数
    pair<string, int> p1("张三", 25);
    cout << p1.first << " " << p1.second << endl;  // 张三 25

    // 方式二：make_pair 工具函数
    pair<string, int> p2 = make_pair("李四", 30);
    cout << p2.first << " " << p2.second << endl;  // 李四 30
}
```

### 2.8 set 的排序规则——仿函数的登场

默认情况下，`set` 按升序排列（使用 `less<T>` 仿函数）。如果你想降序排列，或者存储自定义类型，就需要用到**仿函数**来指定排序规则：

```cpp
// 仿函数：定义一个"大于"规则
class MyGreater {
public:
    bool operator()(int a, int b) const {
        return a > b;                  // 降序
    }
};

void TestSetSort() {
    // 默认升序
    set<int> s1;
    s1.insert(10);
    s1.insert(30);
    s1.insert(20);
    cout << "默认升序: ";
    for (set<int>::iterator it = s1.begin(); it != s1.end(); ++it) {
        cout << *it << " ";            // 10 20 30
    }
    cout << endl;

    // 使用仿函数指定降序
    set<int, MyGreater> s2;
    s2.insert(10);
    s2.insert(30);
    s2.insert(20);
    cout << "指定降序: ";
    for (set<int, MyGreater>::iterator it = s2.begin(); it != s2.end(); ++it) {
        cout << *it << " ";            // 30 20 10
    }
    cout << endl;
}
```

### 2.9 自定义类型存入 set

当你把自定义类型（比如 `Person`）存入 `set` 时，必须告诉它"两个 Person 怎么比大小"。否则编译器无法确定元素的位置：

```cpp
#include <string>

class Person {
public:
    string m_Name;
    int m_Age;

    Person(string name, int age) : m_Name(name), m_Age(age) {}
};

// 指定排序规则：按年龄降序
class CompareByAge {
public:
    bool operator()(const Person& p1, const Person& p2) const {
        return p1.m_Age > p2.m_Age;
    }
};

void TestSetCustomType() {
    set<Person, CompareByAge> persons;

    persons.insert(Person("刘备", 23));
    persons.insert(Person("关羽", 27));
    persons.insert(Person("张飞", 25));
    persons.insert(Person("赵云", 21));

    for (set<Person, CompareByAge>::iterator it = persons.begin();
         it != persons.end(); ++it) {
        cout << "姓名: " << it->m_Name 
             << "  年龄: " << it->m_Age << endl;
    }
}
```

输出：
```
姓名: 关羽  年龄: 27
姓名: 张飞  年龄: 25
姓名: 刘备  年龄: 23
姓名: 赵云  年龄: 21
```

---

## 三、map 容器：键值对的字典

### 3.1 map 的本质

`map` 可以理解为一个自动排序的字典。你通过"键"（key）来快速检索对应的"值"（value）。所有元素都是 `pair<key, value>`，按 key 自动排序。

现实生活中有大量"键值对"场景：身份证号→个人信息、学号→成绩、英文单词→中文释义——`map` 就是对这类场景的完美建模。

### 3.2 构造与赋值

```cpp
#include <iostream>
#include <map>
using namespace std;

void PrintMap(const map<int, int>& m) {
    for (map<int, int>::const_iterator it = m.begin(); 
         it != m.end(); ++it) {
        cout << "key=" << it->first 
             << "  value=" << it->second << endl;
    }
    cout << endl;
}

void TestMapConstruct() {
    map<int, int> m;                       // 默认构造
    m.insert(pair<int, int>(1, 100));
    m.insert(pair<int, int>(3, 300));
    m.insert(pair<int, int>(2, 200));
    PrintMap(m);
    // key=1  value=100
    // key=2  value=200
    // key=3  value=300
    // 注意：自动按 key 排序

    map<int, int> m2(m);                   // 拷贝构造
    PrintMap(m2);

    map<int, int> m3;
    m3 = m;                                // 赋值
    PrintMap(m3);
}
```

### 3.3 四种插入方式

`map` 的插入方式是它最丰富的接口之一，有四种写法。它们的本质相同，但使用场景略有不同：

```cpp
void TestMapInsert() {
    map<int, string> m;

    // 方法一：pair 构造函数
    m.insert(pair<int, string>(1, "苹果"));

    // 方法二：make_pair（最简洁，推荐）
    m.insert(make_pair(2, "香蕉"));

    // 方法三：map 内部的 value_type
    m.insert(map<int, string>::value_type(3, "橙子"));

    // 方法四：[] 运算符（注意：这不是插入，是"访问或创建"）
    m[4] = "葡萄";

    for (map<int, string>::iterator it = m.begin(); 
         it != m.end(); ++it) {
        cout << it->first << " => " << it->second << endl;
    }
}
```

> **`[]` 运算符的陷阱**：`m[key]` 如果 key 不存在，会**自动创建**一个以 key 为键、value 为默认值的元素。如果你只是想知道 key 是否存在，应该用 `find` 而不是 `[]`。很多 bug 就是源于误用 `[]` 无意中插入了垃圾数据。

四种方式的对比：

| 方式 | 优点 | 注意点 |
|------|------|--------|
| `pair<T1,T2>(k,v)` | 最直观 | 需要写类型 |
| `make_pair(k,v)` | 简洁，自动推导类型 | 推荐日常使用 |
| `value_type(k,v)` | 类型明确 | 写法较长 |
| `m[k] = v` | 最简洁 | key 不存在时会自动创建，有副作用 |

### 3.4 大小、交换、查找与统计

```cpp
void TestMapOps() {
    map<int, string> m;
    m.insert(make_pair(1, "张三"));
    m.insert(make_pair(2, "李四"));
    m.insert(make_pair(3, "王五"));

    // 大小
    cout << "元素个数: " << m.size() << endl;     // 3
    cout << "是否为空: " << m.empty() << endl;    // 0

    // 查找
    map<int, string>::iterator it = m.find(2);
    if (it != m.end()) {
        cout << "key=2 的 value: " << it->second << endl; // 李四
    }

    it = m.find(5);
    if (it == m.end()) {
        cout << "key=5 不存在" << endl;
    }

    // 统计：对于 map，count 结果也是 0 或 1（不允许重复 key）
    cout << "key=2 出现次数: " << m.count(2) << endl;  // 1
}
```

### 3.5 删除元素

```cpp
void TestMapErase() {
    map<int, string> m;
    m.insert(make_pair(1, "张三"));
    m.insert(make_pair(2, "李四"));
    m.insert(make_pair(3, "王五"));
    m.insert(make_pair(4, "赵六"));

    m.erase(m.begin());                    // 按迭代器删除
    m.erase(3);                            // 按 key 删除

    for (map<int, string>::iterator it = m.begin();
         it != m.end(); ++it) {
        cout << it->first << " => " << it->second << endl;
    }
    // 2 => 李四
    // 4 => 赵六
}
```

### 3.6 map 的排序规则

与 `set` 同理，`map` 默认按 key 升序。需要自定义顺序时同样使用仿函数：

```cpp
class MyKeyGreater {
public:
    bool operator()(int k1, int k2) const {
        return k1 > k2;                    // 按键降序
    }
};

void TestMapCustomSort() {
    map<int, string, MyKeyGreater> m;
    m.insert(make_pair(3, "张三"));
    m.insert(make_pair(1, "李四"));
    m.insert(make_pair(2, "王五"));

    for (map<int, string, MyKeyGreater>::iterator it = m.begin();
         it != m.end(); ++it) {
        cout << it->first << " => " << it->second << endl;
    }
    // 3 => 张三
    // 2 => 王五
    // 1 => 李四
}
```

---

## 四、函数对象（仿函数）：让操作可定制

### 4.1 什么是函数对象

**函数对象**（Function Object），也称**仿函数**（Functor），是指重载了 `operator()` 的类。它的对象在语法上可以像函数一样被"调用"，但它本质是一个对象——这意味着它可以拥有自己的状态。

```cpp
class MyAdd {
public:
    int operator()(int a, int b) {
        return a + b;
    }
};

void TestFunctor() {
    MyAdd add;
    int result = add(3, 5);               // 看起来像函数调用
    cout << "3 + 5 = " << result << endl;  // 8
}
```

为什么不直接用普通函数？仿函数有三个普通函数无法比拟的优势：

**优势一：可以携带状态**

```cpp
class Counter {
public:
    Counter() : m_Count(0) {}

    void operator()(const string& msg) {
        cout << msg << endl;
        m_Count++;                         // 记录调用次数
    }

    int GetCount() const { return m_Count; }

private:
    int m_Count;                           // 内部状态
};

void TestFunctorState() {
    Counter counter;
    counter("Hello");                      // 调用 3 次
    counter("World");
    counter("C++");
    cout << "调用了 " << counter.GetCount() 
         << " 次" << endl;                 // 调用了 3 次
}
```

普通函数无法记录"自己被调用了多少次"——除非使用丑陋的全局变量或静态局部变量。仿函数则可以通过成员变量优雅地保存状态。

**优势二：可以作为参数传递给算法**

```cpp
#include <vector>
#include <algorithm>

class PrintElement {
public:
    void operator()(int val) const {
        cout << val << " ";
    }
};

void TestFunctorAsParam() {
    vector<int> v;
    for (int i = 0; i < 5; i++) {
        v.push_back(i);
    }

    // 将仿函数对象作为参数传给 for_each
    for_each(v.begin(), v.end(), PrintElement());
    cout << endl;                          // 0 1 2 3 4
}
```

**优势三：编译器更容易内联优化**

仿函数的调用在编译期就能确定（不像函数指针需要运行时解析），因此编译器可以轻松将 `operator()` 内联（inline），消除函数调用的开销。

### 4.2 谓词：返回 bool 的特殊仿函数

**谓词**（Predicate）就是返回值类型为 `bool` 的仿函数。它在 STL 算法中扮演"判定条件"的角色——判断一个元素是否符合某个条件。

- **一元谓词**：`operator()` 接受 1 个参数
- **二元谓词**：`operator()` 接受 2 个参数

#### 一元谓词

```cpp
#include <vector>
#include <algorithm>

// 一元谓词：判断一个数是否大于 5
class GreaterThanFive {
public:
    bool operator()(int val) const {
        return val > 5;
    }
};

void TestUnaryPredicate() {
    vector<int> v;
    for (int i = 1; i <= 10; i++) {
        v.push_back(i);
    }

    // find_if 使用一元谓词查找第一个大于 5 的元素
    vector<int>::iterator it = 
        find_if(v.begin(), v.end(), GreaterThanFive());

    if (it != v.end()) {
        cout << "第一个大于 5 的元素是: " << *it << endl;  // 6
    }
}
```

#### 二元谓词

```cpp
// 二元谓词：比较两个数，用于降序排序
class Descending {
public:
    bool operator()(int a, int b) const {
        return a > b;
    }
};

void TestBinaryPredicate() {
    vector<int> v;
    v.push_back(10);
    v.push_back(40);
    v.push_back(20);
    v.push_back(50);
    v.push_back(30);

    // sort 使用二元谓词改变排序策略
    sort(v.begin(), v.end(), Descending());

    for (vector<int>::iterator it = v.begin(); it != v.end(); ++it) {
        cout << *it << " ";                // 50 40 30 20 10
    }
    cout << endl;
}
```

### 4.3 内建函数对象

不用每次都自己写仿函数——STL 在 `<functional>` 头文件中预置了一批常用仿函数，分为三大类：

#### 算术仿函数

模拟基本运算。其中 `negate` 是一元运算（取反），其余都是二元运算：

```cpp
#include <functional>

void TestArithmeticFunctor() {
    plus<int>        add;       // 加法
    minus<int>       sub;       // 减法
    multiplies<int>  mul;       // 乘法
    divides<int>     div_op;    // 除法
    modulus<int>     mod;       // 取模
    negate<int>      neg;       // 取反（一元）

    cout << "10 + 5 = " << add(10, 5) << endl;       // 15
    cout << "10 - 5 = " << sub(10, 5) << endl;       // 5
    cout << "10 * 5 = " << mul(10, 5) << endl;       // 50
    cout << "10 / 5 = " << div_op(10, 5) << endl;    // 2
    cout << "10 % 5 = " << mod(10, 5) << endl;       // 0
    cout << "-10 取反 = " << neg(10) << endl;         // -10
}
```

#### 关系仿函数

用于比较大小，配合算法改排序策略时最常用：

```cpp
#include <algorithm>

void TestRelationalFunctor() {
    vector<int> v;
    v.push_back(10);
    v.push_back(30);
    v.push_back(50);
    v.push_back(20);
    v.push_back(40);

    // 使用内建的 greater<int> 实现降序排序
    sort(v.begin(), v.end(), greater<int>());

    for (vector<int>::iterator it = v.begin(); it != v.end(); ++it) {
        cout << *it << " ";                // 50 40 30 20 10
    }
    cout << endl;
}
```

内建关系仿函数一览：`equal_to`、`not_equal_to`、`greater`、`greater_equal`、`less`、`less_equal`。六大运算符都有对应。其中 `greater<T>` 和 `less<T>` 使用频率最高——排序时指定用 `greater<int>()` 比手写一个仿函数简洁多了。

#### 逻辑仿函数

实现逻辑运算，在需要做布尔数组变换时偶尔用到：

```cpp
void TestLogicalFunctor() {
    vector<bool> v;
    v.push_back(true);
    v.push_back(false);
    v.push_back(true);
    v.push_back(false);

    cout << "原始: ";
    for (size_t i = 0; i < v.size(); i++) {
        cout << v[i] << " ";               // 1 0 1 0
    }
    cout << endl;

    // 用 transform 配合 logical_not 做逻辑取反
    vector<bool> result;
    result.resize(v.size());
    transform(v.begin(), v.end(), result.begin(), 
              logical_not<bool>());

    cout << "取反: ";
    for (size_t i = 0; i < result.size(); i++) {
        cout << result[i] << " ";          // 0 1 0 1
    }
    cout << endl;
}
```

---

## 五、综合实战：员工分组系统

下面用一个完整的案例把本文知识串联起来。场景：公司招聘了 10 名员工，需要随机分配部门（策划/美术/研发），并按照部门分组显示。

```cpp
#include <iostream>
#include <vector>
#include <map>
#include <string>
#include <ctime>
using namespace std;

// 部门编号
const int CEHUA  = 0;
const int MEISHU = 1;
const int YANFA  = 2;

// 员工类
class Worker {
public:
    string m_Name;
    int m_Salary;
};

// 创建 10 名员工
void CreateWorkers(vector<Worker>& workers) {
    string nameSeed = "ABCDEFGHIJ";
    for (int i = 0; i < 10; i++) {
        Worker w;
        w.m_Name = string("员工") + nameSeed[i];
        w.m_Salary = rand() % 10000 + 10000;  // 10000~19999
        workers.push_back(w);
    }
}

// 随机分配部门
void AssignDepartment(const vector<Worker>& workers,
                      multimap<int, Worker>& departmentMap) {
    for (vector<Worker>::const_iterator it = workers.begin();
         it != workers.end(); ++it) {
        int deptId = rand() % 3;             // 随机 0/1/2
        departmentMap.insert(make_pair(deptId, *it));
    }
}

// 按部门显示
void ShowByDepartment(const multimap<int, Worker>& deptMap, 
                      int deptId, const string& deptName) {
    cout << "\n=== " << deptName << " ===" << endl;

    int count = deptMap.count(deptId);       // 统计该部门人数
    multimap<int, Worker>::const_iterator pos = deptMap.find(deptId);

    for (int i = 0; pos != deptMap.end() && i < count; ++pos, ++i) {
        cout << "姓名: " << pos->second.m_Name
             << "  工资: " << pos->second.m_Salary << endl;
    }
}

int main() {
    srand((unsigned int)time(NULL));

    // 1. 创建员工
    vector<Worker> workers;
    CreateWorkers(workers);

    // 2. 用 multimap 实现部门分组（key=部门编号, value=员工）
    multimap<int, Worker> deptMap;
    AssignDepartment(workers, deptMap);

    // 3. 按部门显示
    ShowByDepartment(deptMap, CEHUA,  "策划部");
    ShowByDepartment(deptMap, MEISHU, "美术部");
    ShowByDepartment(deptMap, YANFA,  "研发部");

    return 0;
}
```

**设计要点赏析**：

- 选用 `multimap` 而非 `map`：因为多个员工可能被分到同一部门，`map` 不允许重复 key
- 利用 `multimap` 的自动排序：部门编号作为 key，自动按 0→1→2 排序
- 利用 `find` + `count` 遍历特定 key 的所有值：先找到第一个匹配的迭代器，再用 `count` 知道该往后走几步

---

## 六、小结

本文完成了 STL 两大板块的收束：

1. **关联式容器**（set、multiset、map、multimap）提供了自动排序和 O(log n) 查找的能力，底层基于红黑树实现。`set` 是单纯的集合，`map` 是键值对字典。它们默认升序排列，可以通过仿函数自定义排序规则。

2. **函数对象（仿函数）** 是 STL 灵活性的关键——它让算法在与不同容器搭配时，仍然可以通过传入的策略对象来改变行为。谓词（返回 bool 的仿函数）和内建仿函数（`greater<T>` 等）是与算法交互最频繁的工具。

掌握了序列式容器和关联式容器，再配合函数对象，你已经可以应对绝大多数 C++ 开发中的数据管理需求。下一篇将深入 STL 的**算法库**——`sort`、`find`、`for_each`、`copy`、`merge`、`set_intersection` 等常用算法的原理与实践。
