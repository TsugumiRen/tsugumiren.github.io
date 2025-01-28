---
title: 一个经典的 diff 算法 —— Myers diff 算法
date: 2025-01-28 20:07:00 +0800
categories: [Algorithm, diff]
tags: [algorithm, dp, greedy]
description: git diff 默认的 Myers 算法
math: true
---

在陵樱学园高中部就读的小镜 (kagami) 同学想要去海下主题公园 LeMU 玩耍。进入这个公园需要登记姓名，很有隐私意识的小镜决定使用假名 tsugumi 来登记。对于每个位置上的字母，小镜可以选择删除、在其后面增加一个字母或是保持不变三个操作，那么小镜应该怎样做才能在尽可能少增删的前提下兼具直观性，把 kagami 变换到 tsugumi 呢？

我们痛恨模棱两可，因此我们不妨对直观性下一个定义，即：在增删操作数相同的情况下，总是选择先删除再增加的操作序列。在定义明晰后，聪明的你肯定已经先笨笨电脑一步看出，我们该怎么变换 kagami 了：

```
-k
-a
+t
+s
+u
 g
-a
+u
 m
 i
```

## LCS 和最小编辑距离

开始一切之前，我们先来简单看看与我们论题相关的两个问题：LCS 和最小编辑距离

LCS (Longest Common Subsequence, 最长公共子序列) 问题是几乎所有算法书在介绍动态规划这一方法时都会介绍的经典算法。其可以描述为：给定字符串 t 和 s，设它们的子序列集分别为 $S$ 和 $T$，求 $S \cap T$ 中最长的子序列。其中某字符串的子序列指的是从该字符串中删除任意个字符后形成的新字符串。

前面忘了后面忘了，但总而言之我们来自于算法课的久远而深邃的记忆告诉我们，LCS 问题的核心在于一个递推方程，能够用上一个子问题的答案来求解下一个规模更大的子问题。用 $f(i,j)\;(0 \leq i \leq m, 0 \leq j \leq n)$ 代表 $S[1..i]$ 和 $T[1..j]$ 的最长子序列长度，那么我们有：

$$
f(i,j)=\begin{cases}
0& i=0\;or\;j=0\\
f(i-1,j-1) + 1 & S[i] = T[j]\\
\max\{f(i-1,j), f(i,j-1)\} & S[i] \neq T[j]
\end{cases}
$$

其很容易理解：

1. 当 $i=0$ 或 $j=0$ 的时候，$S[1..0]$ 或 $T[1..0]$ 代表一个空字符串，那么两者之间的最长公共子序列长度必为 0；
2. 当 $S[i]=T[j]$ 时，我们可以有一个新的公共字符加到 $S[1..i-1]$ 和 $T[1..j-1]$  的最长公共子序列之后，因此新的最长公共子序列比原来的长 1；
3. 当 $S[i]\ne T[j]$ 时，我们没有办法加新字符，因此只能选择已有项中比较长的那一个来作为本位置的最长公共子序列长度。

对于 kagami 和 tsugumi，我们有下面的图来表示各位置 $f(i,j)$ 的值。其中标有颜色的代表一条可能的 $f(i,j)$ 迭代的路径，路径中的蓝色代表该位置的字符属于最长公共子序列，其他颜色则反之。这里 $S\;=\; kagami$，$T\;=\; tsugumi$.

![lcs_matrix](/assets/img/attachments/myers_lcs.png)

动用我们出色的注意力 (Attention is all you need!) ，我们不难发现，如果：

1. 当前位置从 $(i-1,j)$ 变动过来，则删除源字符串位置 $i$ 的字符（红色）；
2. 当前位置从 $(i,j-1)$ 变动过来，则添加目标字符串位置 $j$ 的字符（绿色）；
3. 当前位置从 $(i-1,j-1)$ 变动过来，则保留源字符串位置 $i$ 的字符（蓝色）。

那么这条路径就代表了一个原字符串到目标字符串的编辑操作序列。

既然我们能够根据 LCS 获得一条编辑路径，那么会不会这条路径恰好对应着 $T$ 到 $S$ 的最短编辑路径呢？在有替换操作的最短编辑距离问题中，答案是否定的；但只要我们禁止替换操作，用一个删除操作和一个增加操作来代替它，那么这条路径就是 $T$ 到 $S$ 的最短编辑路径，同时我们有以下公式(med 是 Minimum Edit Distance 的缩写)，其蕴含 LCS 和 MED 之间等价。

$$
med(S,T) = m + n - 2 \times lcs(T,S)
$$

得到这个公式很简单，我们总是可以通过 $m$ 个对 $S$ 的删除操作和 $n$ 个基于 $T$ 的添加操作来将 $S$ 变换到 $T$，在这些删除和添加操作中，各自有 $lcs(T,S)$ 的操作是不必要的——并且，我们也不可能保留更多的字符了，否则 $S$ 要变换到 $T$ 必然至少增加一次删除和添加操作。

到这里，我们似乎已经有了一个可以使用的 diff 算法，只要我们在最后选择编辑路径时，总是选择具有“先删除再增加”序列的编辑路径就行。遗憾的是，dp 算法在干的事情就是填表，尽管可以利用各种方法来优化算法运行时的空间复杂度，但是无论怎么样，我们概念上的表空间大小是 $\mathcal{O}(mn)$，每步填表操作的时间复杂度是 $\mathcal{O}(1)$，于是最后的算法时间复杂度就会是 $\mathcal{O}(mn)$。

## Myers diff 算法

Eugene W.Myers 在 1986 年的 《An O(ND) Difference Algorithm and Its Variations》 一文中提出了一种基于广度优先搜索和贪心策略的 diff 算法。其时间复杂度为 $\mathcal{O}(ND)$，$N$ 指代我们上文中的 $m+n$，而 $D$ 则指最短编辑距离 $med(S,T)$。由于我们用于比较的字符串一般都具有较小的 $D$，因此在这种情况下，Myers diff 算法的时间复杂度优于传统的 dp 方法。

### K-D 坐标系

在 Myers 算法中，我们将定义两种坐标系，一种是 $X-Y$ 坐标系，回忆我们刚才关于 LCS 的递推式，里面的 $i$ 就是 $X$ 坐标，$j$ 则是 $Y$ 坐标。

另外一种坐标系是 $K-D$ 坐标系，其中 $k = x - y$， $d$ 则是我们在广度优先搜索中所经历的步数，在后文你会明白其含义。$k$ 值相等的所有点组成了原来的图中的一条对角线，而 $d$ 值相等的所有点则会形成一条类似等高线的折线，其在我们遍历的每一步中向外扩张，直到折线的一点触及到我们的终点$(x,y)=(m,n)$

懒得画图了，凑合看看原论文里的吧，这里斜着的就是 $k$ 值相同的所有点组成的，粗虚线则是 $d$ 值相同的所有点组成的。

![k_d_figure](/assets/img/attachments/myers-k-d.png)

在论文中 Myers 也证明了，所有步数为 $d$ 的位置，都只可能从 $k=\{-d, -d + 2, ..., d-2,d\}$ 的位置变动而来。

### 搜索过程

1. 对于 $d = \{0, ..., m+n\}$， 进行一步搜索
2. 在每步搜索中，其依次遍历 $k=\{-d, -d + 2, ..., d-2,d\}$，其中有一数组 $V[k] (k=-n...m)$ ：
    1. 计算 $k_{prev}$，其计算公式如下：
        $$
        k_{prev} = \begin{cases}
        k + 1&k=-d\;or\\
        &k\neq d\;and\;V[k+1] > V[k-1]\\
        k-1&
        k=d\;or\\
        &k\neq -d\;and\;V[k+1] \leq V[k-1]\\
        \end{cases}
        $$
     1. 获取上一步的终止位置： $x_{prev}=V[k_{prev}]$，$y_{prev}=x-k_{prev}$
     2. 根据上一步的终止位置获得当前步的起始位置，如果从 $k+1$ 变动过来，那么 $x=x_{prev}$，反之则$x=x_{prev}+1$
     3. 如果$x$和$y$满足终止条件($x=m,y=n$)，则终止；否则，直到$S[x]\neq T[y]$，令$x = x + 1, y = y +1$
     4. 记录当前$(k,d)$下最远到达的位置，令$V[k]=x$

在计算 $k_{prev}$ 时，对于 $k=-d$ 的情况，是因为它只可能从上方变动过来；相反的，对于 $k=d$，其只可能从下方变动过来；对于剩余的情况，我们挑选 $V$ 值较大的一个，因为这能确保我们从较大的 $x$ 值变动过来，从而更接近右下的目标。

获取当前步的位置时，我们会对上一步的 $x$ 做处理以获得新坐标，从下图中不难发现我们为什么这么做。

![k_transition](/assets/img/attachments/myers_k_trasition.svg)

还记得我们上文说过的先删除后增加更符合人类直觉吗，我们从 $k=-d$ 开始遍历就是因为这个原因，先对这条 $k$ 线做处理就等价于先进行删除操作。

### 代码实现

如果上面的文字叙述让你有点迷糊，不妨直接看代码吧：

```cpp
// myers_diff.hpp
#ifndef MYERS_DIFF
#define MYERS_DIFF
#include <deque>
#include <iostream>
#include <ostream>
#include <string>
#include <unordered_map>
namespace diff {
class Myers {
public:
    struct EditStep {
        enum class EditType {
            ADD = 1,
            DELETE,
            KEEP,
        };
        EditType type;
        std::string content;
        EditStep(EditType t, std::string c) :type(t), content(c) {}
    };
    using str = std::string;
    using d_k_pair_t = std::pair<int, int>;
    using edit_path_t = std::deque<EditStep>;
    using v_container = std::unordered_map<int, int>;
    using trace_container = std::unordered_map<int, std::unordered_map<int, int>>;

private:
    str s, t;
    v_container V; // <k, x>, 当前 d 值下各 k 线能达到的最远位置的 x
    // 回溯结果用
    trace_container Vs; // <d, <k, x>>, 各 d 值下的 V
    trace_container Prev; // <d, <k, k_prev>> (d,k) 对应的 (d-1,k'), 上一个状态
    int res_d, res_k;
    bool is_solved = false;

public:
    Myers(str _s, str _t) :s(_s), t(_t) {}

    // 求解问题
    d_k_pair_t solve() {
        if (is_solved) {
            return { res_d, res_k };
        }

        V[1] = 0; // 为了 d = 0 的起始情况赋的值

        int m = s.size(), n = t.size();
        int max_d = m + n;
        for (int d = 0; d < max_d; ++d) { // 因为我们最多只会走 m + n 步
            for (int k = -d; k <= d; k += 2) {
                bool is_from_up = false; // 是否从 k + 1 变动而来
                if (k == -d || (k != d && V[k + 1] > V[k - 1])) {
                    is_from_up = true;
                }
                int k_prev = is_from_up ? k + 1 : k - 1;
                Prev[d][k] = k_prev;

                int x_prev = V[k_prev];
                int x = is_from_up ? x_prev : x_prev + 1;
                int y = x - k;

                // 如果相等, 则前进到不能再前进
                while (x < m && y < n && s[x] == t[y]) {
                    x = x + 1;
                    y = y + 1;
                }
                V[k] = x;
                Vs[d][k] = x;

                if (x == m && y == n) {
                    res_d = d, res_k = k;
                    is_solved = true;
                    return { d, k };
                }
            }
        }

        return { max_d, 0 };
    }

    // 回溯结果以获得编辑路径
    edit_path_t get_path() {
        edit_path_t path;
        auto [d, k] = solve();
        int x = Vs[d][k], y = x - k;
        while (d > 0) {
            int k_prev = Prev[d][k]; // 寻找上一步的 k
            int x_prev = Vs[d-1][k_prev], y_prev = x_prev - k_prev; // 获取上一步的 x, y
            while (x > x_prev && y > y_prev) { // 如果 x 和 y 不与上一步的 x 或 y 相等, 则说明有相同的字符
                path.emplace_front(EditStep::EditType::KEEP, std::string(1, s[x - 1]));
                --x, --y;
            }
            if (x_prev == x) { // 由上方变动而来, 增加
                path.emplace_front(EditStep::EditType::ADD, std::string(1, t[y - 1]));
            }
            else if (y_prev == y) { // 由左方变动而来, 删除
                path.emplace_front(EditStep::EditType::DELETE, std::string(1, s[x - 1]));
            }
            k = k_prev;
            x = x_prev, y = y_prev;
            --d;
        }
        return path;
    }
};

struct DiffPrinter {
    enum class Color {
        DEFAULT = 1,
        GREEN,
        RED,
    };
    static void println_with_color(std::ostream& os, Color color, std::string s) {
        switch (color) {
        case Color::DEFAULT:
            os << s << "\n";
            break;
        case Color::RED:
            os << "\033[0;31m" << s << "\033[0m\n";
            break;
        case Color::GREEN:
            os << "\033[0;32m" << s << "\033[0m\n";
            break;
        }
    }

    static void print_diff(std::ostream& os, Myers::edit_path_t path) {
        using Type = Myers::EditStep::EditType;
        for (const auto& p : path) {
            switch (p.type) {
            case Type::ADD:
                println_with_color(os, Color::GREEN, "+\t" + p.content);
                break;
            case Type::DELETE:
                println_with_color(os, Color::RED, "-\t" + p.content);
                break;
            case Type::KEEP:
                println_with_color(os, Color::DEFAULT, "\t" + p.content);
            }
        }
    }
};
}
#endif

// test.cc
#include "myers_diff.hpp"
int main(){
    std::string s = "kagami", t = "tsugumi";
    diff::Myers m(s,t);
    diff::DiffPrinter::print_diff(std::cout, m.get_path());
    return 0;
}
```

看看我们的结果：

![result](/assets/img/attachments/myers_result.png)

小镜终于能去 LeMU 玩了，不赖。

## 参考

1. [The Myers Diff Algorithm Part 1](https://blog.jcoglan.com/2017/02/12/the-myers-diff-algorithm-part-1/)
2. An O(ND) Difference Algorithm and Its Variations