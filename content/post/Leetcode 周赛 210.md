---
title: Leetcode 周赛 210
date: 2020-10-21
tags:
    - Leetcode
---

# A 括号的最大嵌套深度

**题目**

如果字符串满足一下条件之一，则可以称之为 有效括号字符串（valid parentheses string，可以简写为 VPS）：

字符串是一个空字符串 “”，或者是一个不为 “(“ 或 “)” 的单字符。 字符串可以写为 AB（A 与 B 字符串连接），其中 A 和 B 都是 有效括号字符串 。 字符串可以写为 (A)，其中 A 是一个 有效括号字符串 。 类似地，可以定义任何有效括号字符串 S 的 嵌套深度 depth(S)：

depth(“”) = 0 depth(A + B) = max(depth(A), depth(B))，其中 A 和 B 都是 有效括号字符串 depth(“(“ + A + “)”) = 1 + depth(A)，其中 A 是一个 有效括号字符串 例如：””、”()()”、”()(()())” 都是 有效括号字符串（嵌套深度分别为 0、1、2），而 “)(“ 、”(()” 都不是 有效括号字符串 。

给你一个 有效括号字符串 s，返回该字符串的 s 嵌套深度 。

**思路**

题目相当于在说有效括号字符串可以由文法`S \rightarrow SS,S \rightarrow (S),S \rightarrow a`生成，因此我们可以发现由这样的文法生成的语言中不存在无效的括号，如`(()`，`(((`，之类的，每一对括号内部的左右括号一定是匹配的，因此我们只需要遍历一遍字符串，在遍历中更新当前的嵌套深度`dep`，遇到左括号`dep++`，遇到右括号`dep--`，最终得到最大嵌套深度即可。

**实现**

```python
class Solution:
    def maxDepth(self, s: str) -> int:
        ans = 0
        temp = 0
        stack = []
        for i in s:
            if i == '(':
                temp += 1
            elif i == ')':
                temp -= 1
            ans = max(ans, temp)
        return ans
```

# B 最大网络秩

**题目**

n 座城市和一些连接这些城市的道路 roads 共同组成一个基础设施网络。每个 roads\[i] = \[ai, bi] 都表示在城市 ai 和 bi 之间有一条双向道路。

两座不同城市构成的 城市对 的 网络秩 定义为：与这两座城市 直接 相连的道路总数。如果存在一条道路直接连接这两座城市，则这条道路只计算 一次 。

整个基础设施网络的 最大网络秩 是所有不同城市对中的 最大网络秩 。

给你整数 n 和数组 roads，返回整个基础设施网络的 最大网络秩 。

**思路**

原题中的数据量并不大，n 最大为 100，因此我们可以暴力枚举网络中的所有点对，求出网络的最大网络秩。

在枚举中，首先我们枚举第一个点，求出与它相邻的边。枚举第二个点时，我们需要考虑这两点间是否连通，如果这两点连通则与第二个点相邻的边中存在一条我们已经访问过的边，需要删去。

**实现**

```python
class Solution:
    def maximalNetworkRank(self, n: int, roads: List[List[int]]) -> int:
        ans = 0
        out = [[] for i in range(n)]
        for i in roads:
            out[i[0]].append(i[1])
            out[i[1]].append(i[0])
        for i in range(n):
            maxn = 0
            for x,y in enumerate(out):
                if x != i:
                    temp = len(y)-1 if i in y else len(y)
                    maxn = max(maxn, temp)
            ans = max(ans, len(out[i])+maxn)
        return ans
```

# C 分割两个字符串得到回文串

**题目**

给你两个字符串 a 和 b ，它们长度相同。请你选择一个下标，将两个字符串都在 相同的下标 分割开。由 a 可以得到两个字符串： aprefix 和 asuffix ，满足 a = aprefix + asuffix ，同理，由 b 可以得到两个字符串 bprefix 和 bsuffix ，满足 b = bprefix + bsuffix 。请你判断 aprefix + bsuffix 或者 bprefix + asuffix 能否构成回文串。

当你将一个字符串 s 分割成 sprefix 和 ssuffix 时， ssuffix 或者 sprefix 可以为空。比方说， s = “abc” 那么 “” + “abc” ， “a” + “bc” ， “ab” + “c” 和 “abc” + “” 都是合法分割。

如果 能构成回文字符串 ，那么请返回 true，否则返回 false 。

请注意， x + y 表示连接字符串 x 和 y 。

**思路**

（听说这题数据很烂，用很菜的方法就可以过…）

题目中的分割字符串即 a 的前缀+b 的后缀，或者 b 的前缀+a 的后缀，因此最终所得到的回文串的长度和 ab 的长度是一致的，而它又是个回文串，所以我们可以注意到的是如果其切割的下标不在字符串中点，那么回文串所靠近中点的一部分回文子串一定来源于 a 或 b 串的中间。

因此我们可以分别求出 ab 串的“中心回文子串”，即一个以原字符串的中点为中点的回文子串。之后可以分为两种情况，回文串中心来源于 a 或来源于 b。

如果来源于 a，我们把 a 串中的回文子串删去，同时将 b 串按照对应的长度切片，那么此时我们只需要两个指针，一个从 a 的左端开始一个从 b 的右端开始向中心遍历，检查两个指针是否相同即可，如果遍历超过一半则表示 ab 串可以通过分割得到回文串。需要注意的是我们也需要以 a 的右端 b 的左端为开始遍历一次。

同样如果来源于 b 也可以依照上面的方法处理。

（这个代码写得相当冗余了…存在大量代码浪费的地方…懒得改了…）

**实现**

```python
class Solution:
    def checkPalindromeFormation(self, a: str, b: str) -> bool:
        # 双指针遍历
        def chk(s1, s2):
            l = 0; r = len(s1)-1
            while l < r and s1[l] == s2[r]:
                l += 1
                r -= 1
            if l >= r:
                return True
            else: return False
        length = len(a)
        # 长度分奇偶讨论
        if length%2:
            # 依照 a 的中心回文子串切片
            mid = (length-1)//2-1
            while mid >= 0 and a[mid] == a[length-mid-1]:
                mid -= 1
            mid += 1
            temp1 = a[:mid]+a[length-mid:]; temp2 = b[:mid]+b[length-mid:]
            if chk(temp1, temp2) or chk(temp1[::-1], temp2[::-1]): return True
            # 依照 b 的中心回文子串切片
            mid = (length-1)//2-1
            while mid >= 0 and b[mid] == b[length-mid-1]:
                mid -= 1
            mid += 1
            temp1 = a[:mid]+a[length-mid:]; temp2 = b[:mid]+b[length-mid:]
            if chk(temp1, temp2) or chk(temp1[::-1], temp2[::-1]): return True
            return False
        else:
            # 依照 a 的中心回文子串切片
            mid = length//2-1
            while mid >= 0 and a[mid] == a[length-mid-1]:
                mid -= 1
            mid += 1
            temp1 = a[:mid]+a[length-mid:]; temp2 = b[:mid]+b[length-mid:]
            if chk(temp1, temp2) or chk(temp1[::-1], temp2[::-1]): return True
            # 依照 b 的中心回文子串切片
            mid = length//2-1
            while mid >= 0 and b[mid] == b[length-mid-1]:
                mid -= 1
            mid += 1
            temp1 = a[:mid]+a[length-mid:]; temp2 = b[:mid]+b[length-mid:]
            if chk(temp1, temp2) or chk(temp1[::-1], temp2[::-1]): return True
            return False
```

# D 统计子树中城市之间最大距离

**题目**

给你 n 个城市，编号为从 1 到 n 。同时给你一个大小为 n-1 的数组 edges ，其中 edges\[i] = \[ui, vi] 表示城市 ui 和 vi 之间有一条双向边。题目保证任意城市之间只有唯一的一条路径。换句话说，所有城市形成了一棵 树 。

一棵 子树 是城市的一个子集，且子集中任意城市之间可以通过子集中的其他城市和边到达。两个子树被认为不一样的条件是至少有一个城市在其中一棵子树中存在，但在另一棵子树中不存在。

对于 d 从 1 到 n-1 ，请你找到城市间 最大距离 恰好为 d 的所有子树数目。

请你返回一个大小为 n-1 的数组，其中第 d 个元素（下标从 1 开始）是城市间 最大距离 恰好等于 d 的子树数目。

请注意，两个城市间距离定义为它们之间需要经过的边的数目。

**思路**

题目数据量很小，n 最大只有 15，可以轻易地想到状压。通过状压来遍历树中的点集的同时需要判断这个点集是否能连通以减少遍历数量。

首先如果点集中只有一个点显然没有用。存在多个点时，可以考虑用并查集的思维进行快速判断。如果点集中两个点相邻即两个点中存在一条边，那么可以合并这两个点，连通分量数减 1。通过初始化将每个点设置为孤立点，连通分量数等于点数，之后遍历边，如果两点相邻则连通分量数减 1，最后判断连通分量数是否为 1。一般来说这样的方法是 NG 的，但在不存在环和重边的树上这样的方法可以进行判断。

判断为子树之后我们需要求出所得子树的直径，可以用 dfs 或 bfs 求（这样的话上面的判断过程也可以浓缩进来），考虑到点数不大，我们也可以暴力枚举点集中的所有点对，求点对最短距中的最长距，即`ans=max(min(dis(i,j))){i,j\in S\land i\neq j}`。考虑到这里要经常用到`dis(i,j)`，因此我们可以先通过 floyd 去求出原树中所有点对间的最短距离。

（比赛的时候想到怎么状压 floyd 了，但奈何图论底子太差，忘了怎么写了…正好这里复习一下）

**实现**

```python
class Solution:
    def countSubgraphsForEachDiameter(self, n: int, edges: List[List[int]]) -> List[int]:
        # Floyd 求出所有点对间的最短路径
        dis = [[0x3f3f3f3f for i in range(n)] for i in range(n)]
        d = [0 for i in range(n-1)]
        for i in range(n): dis[i][i] = 0
        for i in edges:
            dis[i[0]-1][i[1]-1] = 1
            dis[i[1]-1][i[0]-1] = 1
        for k in range(n):
            for i in range(n):
                for j in range(n):
                    dis[i][j] = min(dis[i][j], dis[i][k] + dis[k][j])
        # 状压暴力枚举
        for i in range(2**n+1):
            cur = 0
            node = []
            while i:
                if i&1:
                    node.append(cur)
                i = i >> 1
                cur += 1
            nodenum = len(node)
            # 如果只有一个节点显然没有长度大于 0 的路径
            if nodenum == 1:
                continue
            # 并查集思维判断是否连通
            # 遍历边，如果边的两端都在点集中说明这两点相邻，连通分量数减一
            # 因为两点间无重边，因此可以通过这样的方式判断
            # 最终如果点集是联通的，那么最终连通分量数为 1，如果不为 1 则表明不连通
            for i in edges:
                if i[0]-1 in node and i[1]-1 in node:
                    nodenum -= 1
            if nodenum == 1:
                # 遍历点集中点对的路径，求出最长路
                maxn = 0
                for i in range(len(node)):
                    for j in range(i+1, len(node)):
                        maxn = max(maxn, dis[node[i]][node[j]])
                d[maxn-1] += 1
        return d
```
