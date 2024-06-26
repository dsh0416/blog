---
title: 为什么没有三面体？
date: 2024-06-26 09:22:00 +0800
tags: [拓扑学, 欧拉, 欧拉示性数]
---

前几天我在桌子上放了一个骰塔，塔里放了一组 DnD 骰子。有个同事走过来看到一个正四面体的骰子问：「你这是三面骰吗？」我一愣，第一反应是，这世界上不应该存在三面体啊。不过一细想，为什么没有三面体确实是一个值得思考的好问题。

## 欧拉示性数

考虑到我们这需要讨论面的数量，第一个会想要用的公式显然是欧拉示性数公式，即：

$$
V-E+F=2
$$

欧拉示性数是一个拓扑不变量，即对于所有和一个球面同胚的多面体（三维空间中的凸多面体），都适用于这个公式。其中 $$V,E,F$$ 分别是顶点、棱、面的个数。带入 $$F$$ 的数字我们得到：

$$
V-E=-1
$$

不过光有这一个公式似乎不足以解释我们的问题。

## 凸正多面体

如果我们简化一下这个问题，由于是骰子，我们可以先考虑三维空间中的凸正多面体。这个问题古希腊人就有所研究。由于每条棱有两个顶点，且在两个面上。我们定义每个面有 $$p$$ 条棱，经过每个顶点会有 $$q$$ 条棱，围绕棱可以得到下面的式子：

$$
pF=2E=qV
$$

带入数字 3，得到：

$$
3p=2E=qV
$$

我们有三个方程，但是有四个未知数，显然这个问题还是不太好解。好消息是我们知道 $$p,q,E,V$$ 都需要是正整数这个条件。我们将 $$V$$ 用 $$q$$ 代换，能得到：

$$
\frac{2E}{q}=-1+E
$$

将等式两边同处以 $$2E$$，得到：

$$
\frac{1}{q}=-\frac{1}{2E}+\frac{1}{2}
$$

不妨将 $$2E$$ 用 $$3p$$ 替代，我们会得到：

$$
\frac{1}{q}+\frac{1}{3p}=\frac{1}{2}
$$

由于，$$p$$ 和 $$q$$ 都是正整数，显然地，$$p$$ 越大，$$q$$ 越小。我们取 $$p$$ 最小值 1，有 $$q = 6$$ ，此时 $$V = 1/2$$ 不满足。同理，我们依次枚举调大 $$p$$ 的值，直到 $$q < 1$$ 时，都找不到满足条件的整数解。因此不存在正三面体。

## 凸多面体

然而我们一开始想讨论的问题是是否存在三面体，而不是加上限定条件的凸正多面体，好像是答非所问了。而且并非所有的骰子都需要是凸正多面体，比如 d10 骰子就不是凸正多面体：

![D10 骰子怎么看也不是正多面体](/assets/images/dice-d10.png)

D10 骰子怎么看也不是正多面体

我们应该有一个更一般的处理三面体问题的结论。

和二维图形至少需要 3 个顶点类似，面为平面的多面体，至少需要 4 个顶点才能够形成有体积的三维图形。 而 4 个顶点构成的凸多边形就是凸四面体。只考虑这个问题实际上比上面的正凸多面体反而更简单了。

## 多面体

等等，好像我们也没有说我们的三面体得是凸的吧？我们知道所有和一个球面同胚的多面体（三维空间中的凸多面体）的欧拉示性数都是 2，但我们并没有说我们需要和球面同胚吧？比如马克杯和甜甜圈的欧拉示性数就是 0。

![拓扑学家常常分不清马克杯和甜甜圈](/assets/images/mug-donuts.jpg)

拓扑学家常常分不清马克杯和甜甜圈

我们能找到更一般地结论证明三面体不存在吧？不能，因为三面体可以存在，下面给出一个反例：

**一个截面为三角形的甜甜圈，它是不是一个三面体？**

很可惜的是「需要 4 个顶点才能够形成有体积的三维图形」的结论并没有发生改变，通过马克杯构造出来的三面体需要环面的参与才能够完成。但如果我们可以接受环面，我们能不能进一步接受曲面？也许问题可以变得更简单一些，比如圆柱体是否可以看成一个三面体？

## 结论

好吧，其实我一开始的想法想当然了，我们可以有三面体，只是… 可能不那么容易做成骰子。

另外如果我们只从做骰子的角度来考虑的话，我们也并不需要三面体，比如我们将两个正四面体贴合在一起，将对称的两侧都标记为相同的数字的话，也是可以实现三面效果的骰子的。但脱离骰子本身而去重新考虑多面体的意义是一个非常不错的拓扑学训练。

## 扩展阅读

- [This pattern breaks, but for a good reason - 3Blue1Brown](https://youtu.be/YtkIWDE36qU?si=3dZG91P4wCDEgMsE)
- [Why this puzzle is impossible - 3Blue1Brown](https://youtu.be/VvCytJvd4H0?si=7QxzQuQHg2EvHFGj)
