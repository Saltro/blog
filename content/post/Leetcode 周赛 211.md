---
title: Leetcode 周赛 211
date: 2020-10-24
tags:
    - Leetcode
---

# A 两个相同字符之间的最长子字符串

**题目**

给你一个字符串 s，请你返回 两个相同字符之间的最长子字符串的长度 ，计算长度时不含这两个字符。如果不存在这样的子字符串，返回 -1 。

子字符串 是字符串中的一个连续字符序列。

**思路**

记录一个存储每个小写字母出现的开始位置的数组，遍历串维护最长子字符串即可。

**实现**

```python
class Solution:
    def maxLengthBetweenEqualCharacters(self, s: str) -> int:
        a = [-1 for i in range(26)]
        ans = -1
        for j,i in enumerate(s):
            if a[ord(i)-ord('a')] != -1:
                ans = max(ans, j-a[ord(i)-ord('a')]-1)
            else:
                a[ord(i)-ord('a')] = j
        return ans
```

# B 执行操作后字典序最小的字符串

**题目**

给你一个字符串 s 以及两个整数 a 和 b 。其中，字符串 s 的长度为偶数，且仅由数字 0 到 9 组成。

你可以在 s 上按任意顺序多次执行下面两个操作之一：

*   累加：将 a 加到 s 中所有下标为奇数的元素上（下标从 0 开始）。数字一旦超过 9 就会变成 0，如此循环往复。例如，s = “3456” 且 a = 5，则执行此操作后 s 变成 “3951”。

*   轮转：将 s 向右轮转 b 位。例如，s = “3456” 且 b = 1，则执行此操作后 s 变成 “6345”。

请你返回在 s 上执行上述操作任意次后可以得到的 字典序最小 的字符串。

如果两个字符串长度相同，那么字符串 a 字典序比字符串 b 小可以这样定义：在 a 和 b 出现不同的第一个位置上，字符串 a 中的字符出现在字母表中的时间早于 b 中的对应字符。例如，”0158” 字典序比 “0190” 小，因为不同的第一个位置是在第三个字符，显然 ‘5’ 出现在 ‘9’ 之前。

**思路**

算是到比较麻烦的题。

首先我们可以看一下这两个操作的性质。轮转操作有一个关键的性质，由于 s 的长度一定为偶数，当轮转长度为偶数时，那么累加操作永远不会施加到下标为偶数的元素上，反之如果轮转长度为奇数，那么每一个元素都可以施加累加操作。

对于累加操作，可以分为对下标为奇数的元素的累加和对下标为偶数的元素的累加，因为累加只能增加 0-9，因此完全可以将这几种情况遍历一遍，预处理出奇数位和偶数位经过若干次累加操作后所能得到的字符串。

之后便可以暴力求解。轮转相当于在一个长度为串 s 和轮转长度 b 的最小公倍数的循环字符串中按步长为 b 进行遍历。之后只要按照特定的位置切分原字符串维护一个字典序最小的答案即可。详见实现。

**实现**

```python
class Solution:
    def findLexSmallestString(self, s: str, a: int, b: int) -> str:
        # 预处理累加操作后的字符串
        d = [['' for i in range(10)] for i in range(10)]
        for i in range(10):
            for j in range(10):
                for z,t in enumerate(s):
                    if z%2:
                        d[i][j] += str((int(t) + a*j) % 10)
                    else:
                        d[i][j] += str((int(t) + a*i) % 10)
        # 暴力遍历
        ans = '9' * len(s)
        # 如果 b 是奇数那么要遍历每一位上的累加操作
        if b%2:
            for i in d:
                for j in i:
                    for t in range(0, (len(s)*b)//gcd(len(s), b), b):
                        ans = min(ans, j[t%len(s):]+j[:t%len(s)]) # t%len(s) 表示切分原字符串的位置
            return ans
        # 如果是 b 是偶数那么只要遍历偶数位上的累加操作
        else:
            for i in d[0]:
                for t in range(0, (len(s)*b)//gcd(len(s), b), b):
                    ans = min(ans, i[t%len(s):]+i[:t%len(s)])
            return ans
```

# C 无矛盾的最佳球队

**题目**

假设你是球队的经理。对于即将到来的锦标赛，你想组合一支总体得分最高的球队。球队的得分是球队中所有球员的分数 总和 。

然而，球队中的矛盾会限制球员的发挥，所以必须选出一支 没有矛盾 的球队。如果一名年龄较小球员的分数 严格大于 一名年龄较大的球员，则存在矛盾。同龄球员之间不会发生矛盾。

给你两个列表 scores 和 ages，其中每组 scores\[i] 和 ages\[i] 表示第 i 名球员的分数和年龄。请你返回 所有可能的无矛盾球队中得分最高那支的分数 。

**思路**

动态规划。先将数组按照年龄和分数重排，`dp[i]`表示在前 i 名中选取球员并且最后一名为第 i 名时所能得到的最大分数。

这里用到了贪心的思想，最后一名为第 i 名时，第 i 名的分数一定是小于或等于其后的同年龄的球员的分数。这里类似 LIS。

至于状态转移直接在前 i 名球员中遍历即可。状态转移方程可以写为`dp[i]=max\{j\lt idp[j]\}+scores[i]`

**实现**

```python
class Solution:
    def bestTeamScore(self, scores: List[int], ages: List[int]) -> int:
        l = []
        for i in range(len(scores)):
            l.append([ages[i], scores[i]])
        l = sorted(l, key=lambda x:(x[0], x[1]))
        ans = 0
        dp = [0 for i in range(len(scores))]
        for i in range(len(dp)):
            maxn = 0
            for j in range(i):
                if l[j][0] == l[i][0] or l[j][1] <= l[i][1]:
                    maxn = max(maxn, dp[j])
            # 状态转移
            dp[i] += maxn + l[i][1]
            ans = max(ans, dp[i])
        return ans
```

# D 带阈值的图的连通性

**题目**

有 n 座城市，编号从 1 到 n 。编号为 x 和 y 的两座城市直接连通的前提是： x 和 y 的公因数中，至少有一个 严格大于 某个阈值 threshold 。更正式地说，如果存在整数 z ，且满足以下所有条件，则编号 x 和 y 的城市之间有一条道路：

*   x % z == 0

*   y % z == 0

*   z > threshold

给你两个整数 n 和 threshold ，以及一个待查询数组，请你判断每个查询 queries\[i] = \[ai, bi] 指向的城市 ai 和 bi 是否连通（即，它们之间是否存在一条路径）。

返回数组 answer ，其中 answer.length == queries.length 。如果第 i 个查询中指向的城市 ai 和 bi 连通，则 answer\[i] 为 true ；如果不连通，则 answer\[i] 为 false 。

**思路**

对于这一类连通性的问题都是对连通分量点集做并查集，因此只需遍历图中的边做并查集处理即可。而根据题意，我们只需要逆向思考遍历因数 z，对于 z 的两个倍数，在他们中做一条边。

但此题中我们很容易发现图中的边数量过多，直接遍历边的话复杂度太高。观察图，我们可以发现，当着眼于连通性时，图中的大量边并无用处，我们只需要将 z 的倍数从小到大连成链即可，因此在合并时只需要将倍数和因数“合并”即可。

最后考虑阈值，我们只需要从 threshold+1 开始向 n 的二分之一遍历即可。

**实现**

```python
class Solution:
    def areConnected(self, n: int, threshold: int, queries: List[List[int]]) -> List[bool]:
        ans = []
        # 如果阈值为 0，那么因数 1 可以让所有数相连
        if threshold < 1:
            for i in queries:
                ans.append(True)
            return ans
        # 并查集
        fa = []
        for i in range(n+1):
            fa.append(i)
        def find(a):
            if a != fa[a]: fa[a] = find(fa[a])
            return fa[a]
        def union(a, b):
            a = find(a)
            b = find(b)
            if a == b: return True
            fa[a] = b
            return False
        # 枚举因数
        for i in range(threshold+1, n//2+1):
            enu = 2*i
            while enu <= n:
                union(i, enu)
                enu += i

        for i in queries:
            if not find(i[0]) or find(i[0]) != find(i[1]): ans.append(False)
            else: ans.append(True)
        return ans
```
