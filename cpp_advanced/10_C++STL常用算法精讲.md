# 10 C++STL常用算法精讲

## 一、算法库全景

### 1.1 算法与容器的"正交关系"

STL 最精妙的设计，莫过于**算法与容器的彻底分离**。算法不依赖任何具体容器——它只通过迭代器来访问数据。这意味着你只需学一次 `sort`，就能用它给 `vector`、`deque`、甚至普通数组排序。

算法分布在三个头文件中：

| 头文件 | 内容范围 |
|--------|---------|
| `<algorithm>` | 最大的算法库：比较、交换、查找、遍历、复制、修改、排序、集合运算…… |
| `<numeric>` | 小型数值算法：累加、填充、内积、相邻差等 |
| `<functional>` | 内建函数对象（仿函数），配合算法使用 |

### 1.2 质变算法 vs 非质变算法

理解这一分类有助于记住算法的"副作用"：

| 类别 | 是否修改数据 | 典型算法 |
|------|------------|---------|
| 非质变算法 | 不修改 | `find`、`count`、`for_each`、`search`、`binary_search` |
| 质变算法 | 会修改 | `copy`、`replace`、`fill`、`sort`、`merge`、`remove` |

一个常用记法：如果你传入 `const_iterator` 编译器还接受，那多半是非质变算法；如果编译器报错，说明算法试图修改数据。

---

## 二、遍历算法

### 2.1 for_each —— 最优雅的循环

`for_each` 遍历指定区间，对每个元素执行一个函数或仿函数。它替代了手写 `for` 循环，语义更清晰：

**函数原型**：
```
for_each(iterator beg, iterator end, _Func);
```

**参数**：`beg` 起始迭代器，`end` 结束迭代器，`_Func` 是一个一元函数或仿函数。

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
using namespace std;

// 方式一：回调普通函数
void PrintElement(int val) {
    cout << val << " ";
}

// 方式二：回调仿函数
class PrintFunctor {
public:
    void operator()(int val) const {
        cout << val << " ";
    }
};

void TestForEach() {
    vector<int> v;
    for (int i = 0; i < 10; i++) {
        v.push_back(i);
    }

    // 使用普通函数
    for_each(v.begin(), v.end(), PrintElement);
    cout << endl;                          // 0 1 2 3 4 5 6 7 8 9

    // 使用仿函数
    for_each(v.begin(), v.end(), PrintFunctor());
    cout << endl;                          // 0 1 2 3 4 5 6 7 8 9
}
```

> **性能细节**：仿函数版本通常比函数指针版本更快，因为编译器可以直接内联 `operator()`，而函数指针需要间接调用。

### 2.2 transform —— 搬运与变换

`transform` 比 `for_each` 进一步——它不只是在区间上执行操作，而是将操作结果写入**另一个容器**：

**函数原型**：
```
transform(iterator beg1, iterator end1, iterator beg2, _Func);
```

**参数**：`beg1/end1` 源区间，`beg2` 目标容器起始位置，`_Func` 对每个元素执行的变换函数。

```cpp
class Double {
public:
    int operator()(int val) const {
        return val * 2;                    // 每个元素翻倍
    }
};

void TestTransform() {
    vector<int> src;
    for (int i = 0; i < 10; i++) {
        src.push_back(i);
    }

    vector<int> dest;
    dest.resize(src.size());               // ★关键：目标容器必须预分配空间

    transform(src.begin(), src.end(), dest.begin(), Double());

    cout << "变换结果: ";
    for (vector<int>::iterator it = dest.begin(); it != dest.end(); ++it) {
        cout << *it << " ";                // 0 2 4 6 8 10 12 14 16 18
    }
    cout << endl;
}
```

> **常见错误**：忘记 `resize` 目标容器。`transform` 不会自动扩容——它只向已存在的内存位置写入数据。如果目标容器为空，运行时会出现未定义行为。

---

## 三、查找算法

### 3.1 find —— 按值查找

`find` 在区间内线性搜索指定值，找到返回该位置的迭代器，找不到返回 `end()`：

**函数原型**：
```
find(iterator beg, iterator end, value);
```

```cpp
void TestFind() {
    vector<int> v;
    for (int i = 1; i <= 10; i++) {
        v.push_back(i);
    }

    vector<int>::iterator it = find(v.begin(), v.end(), 7);
    if (it != v.end()) {
        cout << "找到: " << *it 
             << " 位置: " << (it - v.begin()) << endl;  // 找到: 7 位置: 6
    }

    it = find(v.begin(), v.end(), 100);
    if (it == v.end()) {
        cout << "未找到 100" << endl;
    }
}
```

**自定义类型使用 find**：需要重载 `operator==`，否则 `find` 不知道两个对象怎样算"相等"：

```cpp
#include <string>

class Student {
public:
    string m_Name;
    int m_Age;
    Student(string name, int age) : m_Name(name), m_Age(age) {}

    // ★ find 依赖此重载
    bool operator==(const Student& other) const {
        return m_Name == other.m_Name && m_Age == other.m_Age;
    }
};

void TestFindCustom() {
    vector<Student> students;
    students.push_back(Student("张三", 20));
    students.push_back(Student("李四", 22));
    students.push_back(Student("王五", 21));

    Student target("李四", 22);
    vector<Student>::iterator it = 
        find(students.begin(), students.end(), target);

    if (it != students.end()) {
        cout << "找到了: " << it->m_Name << endl;  // 李四
    }
}
```

### 3.2 find_if —— 按条件查找

当查找条件不是"等于某个值"而是"满足某个条件"时，用 `find_if` 搭配**一元谓词**：

**函数原型**：
```
find_if(iterator beg, iterator end, _Pred);
```

```cpp
// 一元谓词：判断一个数是否能被 3 整除
class DivisibleBy3 {
public:
    bool operator()(int val) const {
        return val % 3 == 0;
    }
};

void TestFindIf() {
    vector<int> v;
    v.push_back(1);
    v.push_back(4);
    v.push_back(7);
    v.push_back(9);
    v.push_back(10);

    vector<int>::iterator it = find_if(v.begin(), v.end(), DivisibleBy3());

    if (it != v.end()) {
        cout << "第一个能被 3 整除的数: " << *it << endl;  // 9
    }
}
```

### 3.3 adjacent_find —— 查找相邻重复

`adjacent_find` 查找相邻的两个相同元素，返回第一对中前一个元素的迭代器：

```cpp
void TestAdjacentFind() {
    vector<int> v;
    v.push_back(1);
    v.push_back(2);
    v.push_back(2);        // ★ 相邻重复
    v.push_back(3);
    v.push_back(4);
    v.push_back(4);        // ★ 相邻重复

    vector<int>::iterator it = adjacent_find(v.begin(), v.end());

    if (it != v.end()) {
        cout << "第一组相邻重复: " << *it 
             << " 和 " << *(it + 1) << endl;  // 2 和 2
    }
}
```

### 3.4 binary_search —— 二分查找

`binary_search` 利用二分法判断**有序区间**中是否存在某个值。返回 `bool`，不返回位置：

```cpp
void TestBinarySearch() {
    vector<int> v;
    for (int i = 0; i < 20; i++) {
        v.push_back(i);
    }
    // v 已经是有序的

    bool found = binary_search(v.begin(), v.end(), 15);
    cout << "15 存在: " << (found ? "是" : "否") << endl;  // 是

    found = binary_search(v.begin(), v.end(), 100);
    cout << "100 存在: " << (found ? "是" : "否") << endl; // 否
}
```

> **注意**：`binary_search` 的前提是区间必须**已排序**。如果传入乱序区间，结果是未定义的。复杂度 O(log n)，远优于 `find` 的 O(n)。

### 3.5 count —— 统计总数

统计指定值在区间中出现的次数：

```cpp
void TestCount() {
    vector<int> v;
    v.push_back(10);
    v.push_back(20);
    v.push_back(10);
    v.push_back(30);
    v.push_back(10);

    int n = count(v.begin(), v.end(), 10);
    cout << "10 出现了 " << n << " 次" << endl;  // 3
}
```

**自定义类型使用 count**：同样需要重载 `operator==`：

```cpp
void TestCountCustom() {
    vector<Student> students;
    students.push_back(Student("张三", 20));
    students.push_back(Student("李四", 22));
    students.push_back(Student("张三", 20));  // 与第一个完全相同

    Student target("张三", 20);
    int n = count(students.begin(), students.end(), target);
    cout << "张三(20岁) 出现了 " << n << " 次" << endl;  // 2
}
```

### 3.6 count_if —— 按条件统计

与 `count` 的差别就是：不是统计等于某个值的个数，而是统计满足某个条件的个数：

```cpp
class LessThan18 {
public:
    bool operator()(int val) const {
        return val < 18;
    }
};

void TestCountIf() {
    vector<int> ages;
    ages.push_back(15);
    ages.push_back(22);
    ages.push_back(17);
    ages.push_back(30);
    ages.push_back(16);

    int n = count_if(ages.begin(), ages.end(), LessThan18());
    cout << "未满 18 岁的人数: " << n << endl;  // 3
}
```

---

## 四、排序算法

### 4.1 sort —— 排序之王

`sort` 是 STL 中使用频率最高的算法之一。底层通常采用内省排序（introsort），结合了快速排序和堆排序的优势：

**函数原型**：
```
sort(iterator beg, iterator end);           // 默认升序
sort(iterator beg, iterator end, _Pred);    // 自定义规则
```

```cpp
void TestSort() {
    vector<int> v;
    v.push_back(30);
    v.push_back(10);
    v.push_back(50);
    v.push_back(20);
    v.push_back(40);

    // 默认升序
    sort(v.begin(), v.end());
    cout << "升序: ";
    for (vector<int>::iterator it = v.begin(); it != v.end(); ++it) {
        cout << *it << " ";                // 10 20 30 40 50
    }
    cout << endl;

    // 使用内建仿函数降序
    sort(v.begin(), v.end(), greater<int>());
    cout << "降序: ";
    for (vector<int>::iterator it = v.begin(); it != v.end(); ++it) {
        cout << *it << " ";                // 50 40 30 20 10
    }
    cout << endl;
}
```

> **sort 的限制**：`sort` 要求迭代器支持随机访问（`it + n`）。因此 `list` 不能使用全局 `sort`——需要用 `list::sort()` 成员函数。

### 4.2 random_shuffle —— 洗牌

将区间内的元素随机打乱顺序。使用前记得设置随机种子：

```cpp
#include <ctime>

class PrintInt {
public:
    void operator()(int val) const {
        cout << val << " ";
    }
};

void TestRandomShuffle() {
    srand((unsigned int)time(NULL));       // ★ 随机种子

    vector<int> v;
    for (int i = 0; i < 10; i++) {
        v.push_back(i);
    }

    cout << "打乱前: ";
    for_each(v.begin(), v.end(), PrintInt());
    cout << endl;                          // 0 1 2 3 4 5 6 7 8 9

    random_shuffle(v.begin(), v.end());
    cout << "打乱后: ";
    for_each(v.begin(), v.end(), PrintInt());
    cout << endl;                          // 某次运行: 4 1 9 2 0 6 7 3 8 5
}
```

> **应用场景**：发牌、随机出题、数据集的随机划分（训练集/测试集）等。

### 4.3 merge —— 合并有序序列

将两个**已排序**的区间合并为一个有序区间：

**函数原型**：
```
merge(beg1, end1, beg2, end2, dest);
```

```cpp
void TestMerge() {
    vector<int> v1;
    vector<int> v2;
    for (int i = 0; i < 6; i++) {
        v1.push_back(i * 2);               // 0 2 4 6 8 10
        v2.push_back(i * 2 + 1);           // 1 3 5 7 9 11
    }
    // 两个序列各自有序

    vector<int> result;
    result.resize(v1.size() + v2.size());  // ★ 预分配

    merge(v1.begin(), v1.end(), 
          v2.begin(), v2.end(), 
          result.begin());

    cout << "合并后: ";
    for_each(result.begin(), result.end(), PrintInt());
    cout << endl;                          // 0 1 2 3 4 5 6 7 8 9 10 11
}
```

> **merge vs sort+insert**：如果两个序列已经有序，`merge` 是 O(n) 的，而"全部插入再 sort"是 O(n log n) 的。知道数据已排序时，用 `merge` 高效得多。

### 4.4 reverse —— 反转区间

将区间内所有元素的顺序颠倒：

```cpp
void TestReverse() {
    vector<int> v;
    v.push_back(1);
    v.push_back(2);
    v.push_back(3);
    v.push_back(4);
    v.push_back(5);

    cout << "反转前: ";
    for_each(v.begin(), v.end(), PrintInt());  // 1 2 3 4 5
    cout << endl;

    reverse(v.begin(), v.end());

    cout << "反转后: ";
    for_each(v.begin(), v.end(), PrintInt());  // 5 4 3 2 1
    cout << endl;
}
```

---

## 五、拷贝与替换算法

### 5.1 copy —— 区间拷贝

将源区间的元素拷贝到目标区间。与 `transform` 一样，目标区间需要预分配空间：

```cpp
void TestCopy() {
    vector<int> src;
    for (int i = 1; i <= 5; i++) {
        src.push_back(i * 10);             // 10 20 30 40 50
    }

    vector<int> dest;
    dest.resize(src.size());               // ★ 预分配

    copy(src.begin(), src.end(), dest.begin());

    cout << "拷贝结果: ";
    for_each(dest.begin(), dest.end(), PrintInt());  // 10 20 30 40 50
    cout << endl;
}
```

> **copy vs assign**：`copy` 是通用算法，可以作用于任何迭代器范围；而大量容器也有自己的 `assign(beg, end)` 方法。对于同一容器类型，两者效果相同；但如果目标容器已经创建好（如已 resize），用 `copy` 更灵活。

### 5.2 replace —— 值替换

将区间中所有等于 `oldValue` 的元素替换为 `newValue`：

**函数原型**：
```
replace(iterator beg, iterator end, oldValue, newValue);
```

```cpp
void TestReplace() {
    vector<int> v;
    v.push_back(10);
    v.push_back(20);
    v.push_back(10);
    v.push_back(30);
    v.push_back(10);

    cout << "替换前: ";
    for_each(v.begin(), v.end(), PrintInt());  // 10 20 10 30 10
    cout << endl;

    replace(v.begin(), v.end(), 10, 100);

    cout << "替换后: ";
    for_each(v.begin(), v.end(), PrintInt());  // 100 20 100 30 100
    cout << endl;
}
```

### 5.3 replace_if —— 条件替换

`replace` 只能按**值**替换，`replace_if` 按**条件**替换——搭配一元谓词使用：

**函数原型**：
```
replace_if(iterator beg, iterator end, _Pred, newValue);
```

```cpp
// 谓词：大于等于 30 的元素
class GreaterEqual30 {
public:
    bool operator()(int val) const {
        return val >= 30;
    }
};

void TestReplaceIf() {
    vector<int> v;
    v.push_back(10);
    v.push_back(20);
    v.push_back(30);
    v.push_back(40);
    v.push_back(50);

    cout << "替换前: ";
    for_each(v.begin(), v.end(), PrintInt());  // 10 20 30 40 50
    cout << endl;

    replace_if(v.begin(), v.end(), GreaterEqual30(), 999);

    cout << "替换后: ";
    for_each(v.begin(), v.end(), PrintInt());  // 10 20 999 999 999
    cout << endl;
}
```

### 5.4 swap —— 容器交换

交换两个**同类型**容器的全部内容。直接交换内部指针，非常高效（O(1)）：

```cpp
void TestSwap() {
    vector<int> v1;
    vector<int> v2;
    for (int i = 0; i < 5; i++) {
        v1.push_back(i);
        v2.push_back(i + 100);
    }

    cout << "交换前:" << endl;
    cout << "v1: "; for_each(v1.begin(), v1.end(), PrintInt()); cout << endl;
    cout << "v2: "; for_each(v2.begin(), v2.end(), PrintInt()); cout << endl;

    swap(v1, v2);                          // 全局 swap 算法

    cout << "交换后:" << endl;
    cout << "v1: "; for_each(v1.begin(), v1.end(), PrintInt()); cout << endl;
    cout << "v2: "; for_each(v2.begin(), v2.end(), PrintInt()); cout << endl;
}
```

---

## 六、算术生成算法

这两个算法在 `<numeric>` 头文件中（不是 `<algorithm>`）。

### 6.1 accumulate —— 累加求和

计算区间内所有元素的总和，可指定初始值：

**函数原型**：
```
accumulate(iterator beg, iterator end, initValue);
```

```cpp
#include <numeric>

void TestAccumulate() {
    vector<int> v;
    for (int i = 1; i <= 100; i++) {
        v.push_back(i);
    }

    int total = accumulate(v.begin(), v.end(), 0);
    cout << "1 到 100 的和 = " << total << endl;  // 5050

    // 初始值不为 0 的情况
    int totalWithBase = accumulate(v.begin(), v.end(), 1000);
    cout << "加基础值后 = " << totalWithBase << endl;  // 6050
}
```

### 6.2 fill —— 批量填充

将区间内所有元素设置为同一个值。即使区间已经有数据也会全部覆盖：

```cpp
void TestFill() {
    vector<int> v;
    v.resize(10);                          // 先分配 10 个位置（默认值为 0）

    fill(v.begin(), v.end(), 42);

    for_each(v.begin(), v.end(), PrintInt());
    cout << endl;                          // 42 42 42 42 42 42 42 42 42 42

    // 可以只填充区间的一部分
    fill(v.begin(), v.begin() + 5, 7);
    for_each(v.begin(), v.end(), PrintInt());
    cout << endl;                          // 7 7 7 7 7 42 42 42 42 42
}
```

---

## 七、集合算法

集合算法处理**有序序列**之间的数学集合运算。三个算法的前提一致：参与运算的两个序列必须是**有序的**。

### 7.1 set_intersection —— 交集

求两个集合的交集（同时属于 A 和 B 的元素）：

**函数原型**：
```
set_intersection(beg1, end1, beg2, end2, dest);
```

> **返回值**：指向目标区间中最后一个交集元素**之后**的位置的迭代器。遍历时应该用这个返回值作为终点而非 `dest.end()`。

```cpp
void TestSetIntersection() {
    vector<int> v1;
    vector<int> v2;
    for (int i = 0; i < 10; i++) {
        v1.push_back(i);                   // 0 1 2 3 4 5 6 7 8 9
        v2.push_back(i + 5);               // 5 6 7 8 9 10 11 12 13 14
    }

    vector<int> result;
    result.resize(min(v1.size(), v2.size()));  // ★ 取较小值分配

    vector<int>::iterator endIt = 
        set_intersection(v1.begin(), v1.end(),
                         v2.begin(), v2.end(),
                         result.begin());

    cout << "交集: ";
    for (vector<int>::iterator it = result.begin(); it != endIt; ++it) {
        cout << *it << " ";                // 5 6 7 8 9
    }
    cout << endl;
}
```

**空间分配规则**：交集元素个数不可能超过两个集合中较小的那个，因此用 `min(size1, size2)` 分配就够了。

### 7.2 set_union —— 并集

求两个集合的并集（属于 A 或属于 B 的所有元素，去重）：

```cpp
void TestSetUnion() {
    vector<int> v1;
    vector<int> v2;
    for (int i = 0; i < 10; i++) {
        v1.push_back(i);                   // 0 1 2 3 4 5 6 7 8 9
        v2.push_back(i + 5);               // 5 6 7 8 9 10 11 12 13 14
    }

    vector<int> result;
    result.resize(v1.size() + v2.size());  // ★ 取两容量之和分配

    vector<int>::iterator endIt = 
        set_union(v1.begin(), v1.end(),
                  v2.begin(), v2.end(),
                  result.begin());

    cout << "并集: ";
    for (vector<int>::iterator it = result.begin(); it != endIt; ++it) {
        cout << *it << " ";                // 0 1 2 3 4 5 6 7 8 9 10 11 12 13 14
    }
    cout << endl;
}
```

**空间分配规则**：并集元素个数最大可能是两个集合大小之和（完全没有重叠时），因此用 `size1 + size2` 分配最安全。

### 7.3 set_difference —— 差集

求两个集合的差集（属于 A 但不属于 B）。**差集不满足交换律**——A 对 B 的差集和 B 对 A 的差集结果不同：

```cpp
void TestSetDifference() {
    vector<int> v1;
    vector<int> v2;
    for (int i = 0; i < 10; i++) {
        v1.push_back(i);                   // 0 1 2 3 4 5 6 7 8 9
        v2.push_back(i + 5);               // 5 6 7 8 9 10 11 12 13 14
    }

    vector<int> result;
    result.resize(max(v1.size(), v2.size()));  // ★ 取较大值分配

    // v1 - v2：在 v1 中但不在 v2 中
    vector<int>::iterator endIt = 
        set_difference(v1.begin(), v1.end(),
                       v2.begin(), v2.end(),
                       result.begin());
    cout << "v1 - v2: ";
    for (vector<int>::iterator it = result.begin(); it != endIt; ++it) {
        cout << *it << " ";                // 0 1 2 3 4
    }
    cout << endl;

    // v2 - v1：在 v2 中但不在 v1 中
    endIt = set_difference(v2.begin(), v2.end(),
                           v1.begin(), v1.end(),
                           result.begin());
    cout << "v2 - v1: ";
    for (vector<int>::iterator it = result.begin(); it != endIt; ++it) {
        cout << *it << " ";                // 10 11 12 13 14
    }
    cout << endl;
}
```

**空间分配规则**：差集元素个数最大可能等于较大的那个集合（较小的集合完全被减去），因此用 `max(size1, size2)` 分配。

### 7.4 三个集合算法的空间分配速记表

| 算法 | 目标空间分配 | 口诀 |
|------|------------|------|
| 交集 | `min(size1, size2)` | 交小 |
| 并集 | `size1 + size2` | 并加 |
| 差集 | `max(size1, size2)` | 差大 |

---

## 八、综合实战：学生成绩分析系统

下面是一个综合案例，将遍历、查找、排序、统计、条件替换等算法融会贯通：

```cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <numeric>
#include <string>
#include <ctime>
using namespace std;

class Student {
public:
    string m_Name;
    int m_Score;

    Student(string name, int score) 
        : m_Name(name), m_Score(score) {}
};

// 打印一名学生
class PrintStudent {
public:
    void operator()(const Student& s) const {
        cout << "  姓名: " << s.m_Name 
             << "  成绩: " << s.m_Score << endl;
    }
};

// 判断是否及格（>=60）
class IsPassed {
public:
    bool operator()(const Student& s) const {
        return s.m_Score >= 60;
    }
};

// 判断是否不及格
class IsFailed {
public:
    bool operator()(const Student& s) const {
        return s.m_Score < 60;
    }
};

// 降序排序（用于 sort）
bool ScoreDescending(const Student& a, const Student& b) {
    return a.m_Score > b.m_Score;
}

int main() {
    srand((unsigned int)time(NULL));

    // 1. 构造学生数据
    vector<Student> students;
    string names[] = { "张三","李四","王五","赵六","孙七",
                       "周八","吴九","郑十","冯一","陈二" };
    for (int i = 0; i < 10; i++) {
        int score = rand() % 101;          // 0~100 随机分数
        students.push_back(Student(names[i], score));
    }

    cout << "========== 全体学生成绩 ==========" << endl;
    for_each(students.begin(), students.end(), PrintStudent());

    // 2. 统计及格人数和不及格人数
    int passedCount = count_if(students.begin(), students.end(), IsPassed());
    int failedCount = count_if(students.begin(), students.end(), IsFailed());
    cout << "\n及格人数: " << passedCount 
         << "  不及格人数: " << failedCount << endl;

    // 3. 计算平均分
    int totalScore = 0;
    for (vector<Student>::iterator it = students.begin();
         it != students.end(); ++it) {
        totalScore += it->m_Score;
    }
    double avgScore = static_cast<double>(totalScore) / students.size();
    cout << "平均分: " << avgScore << endl;

    // 4. 查找第一个满分学生
    vector<Student>::iterator perfectIt = 
        find_if(students.begin(), students.end(), IsPassed());
    // 更精确地找满分：需要自定义谓词；此处简化为找第一个及格的
    if (perfectIt != students.end()) {
        cout << "第一个及格的学生: " << perfectIt->m_Name 
             << " (" << perfectIt->m_Score << "分)" << endl;
    }

    // 5. 按成绩降序排序
    sort(students.begin(), students.end(), ScoreDescending);
    cout << "\n========== 按成绩降序排名 ==========" << endl;
    for_each(students.begin(), students.end(), PrintStudent());

    // 6. "成绩美化"：将不及格的分数标记为 0（实际中不会这样，仅演示 replace_if）
    // 先拷贝一份用于演示
    vector<Student> copyStudents(students);
    replace_if(copyStudents.begin(), copyStudents.end(), IsFailed(), 
               Student("", 0));  // 这里只为演示，实际使用需注意

    cout << "\n========== 演示 replace_if（不及格置0） ==========" << endl;
    for_each(copyStudents.begin(), copyStudents.end(), PrintStudent());

    return 0;
}
```

这个案例串起了 `for_each`、`count_if`、`find_if`、`sort`、`replace_if` 五种算法，展示了 STL 算法在实际业务中的组合使用方式。

---

## 九、常用算法速查表

| 算法 | 头文件 | 功能 | 关键前提/注意 |
|------|--------|------|-------------|
| `for_each` | `<algorithm>` | 遍历 | 仿函数版本有性能优势 |
| `transform` | `<algorithm>` | 搬运变换 | **目标容器必须预分配空间** |
| `find` | `<algorithm>` | 按值查找 | 自定义类型需重载 `==` |
| `find_if` | `<algorithm>` | 条件查找 | 搭配一元谓词使用 |
| `adjacent_find` | `<algorithm>` | 相邻重复查找 | 返回第一对的第一个位置 |
| `binary_search` | `<algorithm>` | 二分查找 | **区间必须有序**，只返回 bool |
| `count` | `<algorithm>` | 按值统计 | 自定义类型需重载 `==` |
| `count_if` | `<algorithm>` | 条件统计 | 搭配一元谓词使用 |
| `sort` | `<algorithm>` | 排序 | 需随机访问迭代器，list 用自身的 `sort()` |
| `random_shuffle` | `<algorithm>` | 随机打乱 | 需先 `srand` 设置种子 |
| `merge` | `<algorithm>` | 合并有序序列 | **两个源区间必须有序**，目标需预分配 |
| `reverse` | `<algorithm>` | 反转 | 面试常见题 |
| `copy` | `<algorithm>` | 拷贝 | **目标容器必须预分配空间** |
| `replace` | `<algorithm>` | 按值替换 | 替换所有匹配值 |
| `replace_if` | `<algorithm>` | 条件替换 | 搭配一元谓词 |
| `swap` | `<algorithm>` | 交换容器 | 两个容器类型必须一致 |
| `accumulate` | `<numeric>` | 累加求和 | 注意头文件是 `<numeric>` |
| `fill` | `<numeric>` | 批量填充 | 覆盖已有数据 |
| `set_intersection` | `<algorithm>` | 交集 | **两个源区间必须有序**，取小分配 |
| `set_union` | `<algorithm>` | 并集 | **两个源区间必须有序**，相加分配 |
| `set_difference` | `<algorithm>` | 差集 | **两个源区间必须有序**，取大分配 |

---

## 十、小结

本文覆盖了 STL 算法库中最常用、最实用的 21 个算法，分为六大类：

1. **遍历**（`for_each`、`transform`）—— 替代手写循环
2. **查找**（`find`、`find_if`、`adjacent_find`、`binary_search`、`count`、`count_if`）—— 数据检索的利器
3. **排序**（`sort`、`random_shuffle`、`merge`、`reverse`）—— 重新排列元素
4. **拷贝与替换**（`copy`、`replace`、`replace_if`、`swap`）—— 修改容器内容
5. **算术生成**（`accumulate`、`fill`）—— 数值运算与批量赋值
6. **集合运算**（`set_intersection`、`set_union`、`set_difference`）—— 有序序列的数学集合

掌握了这些算法，你就已经覆盖了 C++ 日常开发中 90% 以上的数据操作需求。再配合前两篇文章所讲的序列式容器和关联式容器，STL 的三大核心（容器、算法、迭代器）已全部到位，你完全可以自信地在项目中使用 STL 了。
