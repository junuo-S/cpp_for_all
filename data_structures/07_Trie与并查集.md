# 07 Trie与并查集

## 一、Trie（前缀树）

### 1.1 为什么需要 Trie？

假设你有一万个英文单词，需要频繁执行：

- 查找某个单词是否存在
- 查找以某前缀开头的所有单词（自动补全）

用哈希表？查找单词 O(1)，但**前缀搜索**需要遍历所有键。用 Trie 可以在 O(L)（L 为单词长度）内完成这两种操作。

### 1.2 Trie 的结构

Trie 是一棵**多叉树**，每条边代表一个字符，从根到某节点的路径构成一个前缀：

```
        (root)
       / | \
      a  b  c
     /       \
    p         a
   / \         \
  p   i        t
  |   |
  l   l
  |   |
  e*  e*
```

存储的单词：`apple`, `api`, `cat`（`*` 标记 isEnd）

> 注：上图为简化示意，实际每个节点只存储**一个字符**，如 "apple" 对应 a→p→p→l→e 共 5 层节点。

关键特征：

- 根节点不包含字符
- 每个节点最多有 26 个孩子（小写字母）
- 用 `isEnd` 标记某节点是否为完整单词的结尾

### 1.3 Trie 的实现

```cpp
#include <iostream>
#include <string>
#include <vector>
using namespace std;

struct TrieNode {
    TrieNode* children[26];
    bool isEnd;

    TrieNode() : isEnd(false) {
        for (int i = 0; i < 26; ++i) children[i] = nullptr;
    }
};

class Trie {
private:
    TrieNode* root;

public:
    Trie() { root = new TrieNode(); }

    // 插入单词 O(L)
    void insert(const string& word) {
        TrieNode* cur = root;
        for (char c : word) {
            int idx = c - 'a';
            if (!cur->children[idx]) {
                cur->children[idx] = new TrieNode();
            }
            cur = cur->children[idx];
        }
        cur->isEnd = true;
    }

    // 查找单词 O(L)
    bool search(const string& word) const {
        const TrieNode* node = findNode(word);
        return node && node->isEnd;
    }

    // 判断是否存在以 prefix 为前缀的单词 O(L)
    bool startsWith(const string& prefix) const {
        return findNode(prefix) != nullptr;
    }

    // 获取所有以 prefix 为前缀的单词
    vector<string> getWordsWithPrefix(const string& prefix) const {
        vector<string> results;
        const TrieNode* node = findNode(prefix);
        if (!node) return results;

        string current = prefix;
        dfs(node, current, results);
        return results;
    }

private:
    const TrieNode* findNode(const string& prefix) const {
        const TrieNode* cur = root;
        for (char c : prefix) {
            int idx = c - 'a';
            if (!cur->children[idx]) return nullptr;
            cur = cur->children[idx];
        }
        return cur;
    }

    void dfs(const TrieNode* node, string& current, vector<string>& results) const {
        if (node->isEnd) {
            results.push_back(current);
        }
        for (int i = 0; i < 26; ++i) {
            if (node->children[i]) {
                current.push_back('a' + i);
                dfs(node->children[i], current, results);
                current.pop_back();
            }
        }
    }
};

int main() {
    Trie trie;
    trie.insert("apple");
    trie.insert("app");
    trie.insert("application");
    trie.insert("banana");

    cout << boolalpha;
    cout << "search(app): " << trie.search("app") << endl;              // true
    cout << "search(ap): " << trie.search("ap") << endl;                // false
    cout << "startsWith(ap): " << trie.startsWith("ap") << endl;        // true

    cout << "以 app 为前缀的单词: ";
    for (const string& w : trie.getWordsWithPrefix("app")) {
        cout << w << " ";
    }
    cout << endl;  // app apple application

    return 0;
}
```

### 1.4 Trie 的复杂度

| 操作 | 时间复杂度 | 说明 |
|------|-----------|------|
| 插入 | O(L) | L 为单词长度 |
| 查找 | O(L) | 与单词数无关 |
| 前缀搜索 | O(L + K) | K 为匹配结果总字符数 |
| 空间 | O(N × L × 26) | N 为单词数（最坏，实际共享前缀节省大量空间） |

### 1.5 Trie 的应用场景

| 场景 | 说明 |
|------|------|
| 自动补全 / 搜索建议 | 输入前缀，返回候选词 |
| 拼写检查 | 判断单词是否在词典中 |
| IP 路由（最长前缀匹配） | 网络路由器查表 |
| 词频统计 | 节点额外存储 count |
| 最大异或值 | 二进制 Trie（01-Trie） |

---

## 二、并查集（Union-Find）

### 2.1 问题引入

有 n 个元素，需要支持：

- **合并（Union）**：将两个元素所在的集合合并
- **查找（Find）**：判断两个元素是否属于同一集合

例如：社交网络中判断两人是否在同一朋友圈、图中判断两点是否连通。

### 2.2 基本思想

用一棵**树**（森林）表示一个集合，树根是集合的代表元素。每个元素记录其父节点：

```
初始：每个元素自成一集合
parent: [0, 1, 2, 3, 4]

合并 0 和 1：
  0
  |
  1
parent: [0, 0, 2, 3, 4]

合并 2 和 3：
  2
  |
  3
parent: [0, 0, 2, 2, 4]

合并 1 和 3（即合并两棵树）：
  0
  |
  1
  |
  2
  |
  3
parent: [0, 0, 0, 0, 4]（简化示意）
```

### 2.3 朴素实现的问题

如果不加优化，合并可能形成一条长链，`find` 退化为 O(n)。

两个关键优化：

| 优化 | 做法 | 效果 |
|------|------|------|
| 路径压缩 | find 时将路径上所有节点直接指向根 | 树变"扁平" |
| 按秩合并 | 将矮树接到高树下 | 控制树高 |

两者结合后，单次操作均摊复杂度为 **O(α(n))**，其中 α 是反阿克曼函数，实际不超过 5。

### 2.4 完整实现

```cpp
#include <iostream>
#include <vector>
#include <numeric>
using namespace std;

class UnionFind {
private:
    vector<int> parent;
    vector<int> rank_;  // 树的近似高度
    int components;     // 连通分量数

public:
    UnionFind(int n) : parent(n), rank_(n, 0), components(n) {
        iota(parent.begin(), parent.end(), 0);  // parent[i] = i
    }

    // 查找（带路径压缩）
    int find(int x) {
        if (parent[x] != x) {
            parent[x] = find(parent[x]);  // 递归压缩
        }
        return parent[x];
    }

    // 合并（按秩）
    bool unite(int x, int y) {
        int rx = find(x);
        int ry = find(y);
        if (rx == ry) return false;  // 已在同一集合

        if (rank_[rx] < rank_[ry]) swap(rx, ry);
        parent[ry] = rx;
        if (rank_[rx] == rank_[ry]) ++rank_[rx];

        --components;
        return true;
    }

    // 判断是否连通
    bool connected(int x, int y) {
        return find(x) == find(y);
    }

    int count() const { return components; }
};

int main() {
    UnionFind uf(5);

    uf.unite(0, 1);
    uf.unite(2, 3);
    uf.unite(1, 3);

    cout << boolalpha;
    cout << "0 和 3 连通: " << uf.connected(0, 3) << endl;  // true
    cout << "0 和 4 连通: " << uf.connected(0, 4) << endl;  // false
    cout << "连通分量数: " << uf.count() << endl;            // 2

    return 0;
}
```

### 2.5 路径压缩的迭代写法

递归写法在极端情况下可能栈溢出，迭代版更安全：

```cpp
int find(int x) {
    // 第一遍：找根
    int root = x;
    while (parent[root] != root) {
        root = parent[root];
    }
    // 第二遍：压缩路径
    while (parent[x] != root) {
        int next = parent[x];
        parent[x] = root;
        x = next;
    }
    return root;
}
```

### 2.6 经典应用：判断无向图的环

```cpp
#include <iostream>
#include <vector>
using namespace std;

class UnionFind {
    vector<int> parent, rank_;
public:
    UnionFind(int n) : parent(n), rank_(n, 0) {
        for (int i = 0; i < n; ++i) parent[i] = i;
    }
    int find(int x) {
        if (parent[x] != x) parent[x] = find(parent[x]);
        return parent[x];
    }
    bool unite(int x, int y) {
        int rx = find(x), ry = find(y);
        if (rx == ry) return false;
        if (rank_[rx] < rank_[ry]) swap(rx, ry);
        parent[ry] = rx;
        if (rank_[rx] == rank_[ry]) ++rank_[rx];
        return true;
    }
};

// 检测无向图是否有环
bool hasCycle(int n, const vector<pair<int, int>>& edges) {
    UnionFind uf(n);
    for (auto& [u, v] : edges) {
        if (!uf.unite(u, v)) {
            return true;  // 两点已连通，再加边就成环
        }
    }
    return false;
}

int main() {
    // 图: 0-1, 1-2, 0-2（三角形 → 有环）
    vector<pair<int, int>> edges1 = {{0,1}, {1,2}, {0,2}};
    cout << boolalpha << hasCycle(3, edges1) << endl;  // true

    // 图: 0-1, 1-2（链 → 无环）
    vector<pair<int, int>> edges2 = {{0,1}, {1,2}};
    cout << hasCycle(3, edges2) << endl;  // false

    return 0;
}
```

### 2.7 经典应用：Kruskal 最小生成树（预览）

Kruskal 算法的核心步骤：

1. 将所有边按权重排序
2. 依次取最小边，若两端不在同一集合则加入（用并查集判断）
3. 直到选了 n-1 条边

> 详细实现见《09 图的经典算法》。

---

## 三、Trie 与并查集对比

| 维度 | Trie | 并查集 |
|------|------|--------|
| 解决的问题 | 字符串前缀匹配 | 集合合并与连通性判断 |
| 数据结构 | 多叉树 | 森林（数组模拟） |
| 核心操作 | insert / search / startsWith | find / unite |
| 时间复杂度 | O(L)（L 为字符串长度） | O(α(n)) ≈ O(1) |
| 典型应用 | 自动补全、词典、IP 路由 | 连通分量、环检测、Kruskal |

---

## 四、总结

| 要点 | 内容 |
|------|------|
| Trie 核心 | 按字符逐层建树，共享前缀节省空间，O(L) 查找/前缀搜索 |
| Trie 局限 | 空间开销大（每个节点 26 个指针），不适合存储非字符串 |
| 并查集核心 | parent 数组 + 路径压缩 + 按秩合并，近 O(1) 合并/查询 |
| 并查集局限 | 只支持合并和查询，不支持分割集合 |
| 工程联系 | Trie 无 STL 直接对应（需手写或用第三方库）；并查集在图算法中广泛使用 |
