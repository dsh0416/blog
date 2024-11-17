---
title: sin(x) = 2 的数学对决
date: 2024-11-18 01:00:00 +0800
tags: [数学, 数学分析]
---

这周末我在微信朋友圈搞了个有趣的数学对决。我给 24 小时内所有解出了这个谜题的人都发了 50 元的红包。题目本身其实不算很难，甚至是一道复分析书上的例题。不过试图使用 LLM 来算这道题的人表现出了很神秘的高度一致的错误...

## 题目

$$
\sin(x) = 2, (x \in \mathbb{C})
$$

### 提示

1. 欧拉公式
2. 注意解的数量超过 1 个
3. 二次方程求根公式

## 题解

$$ sin(x) $$ 在实数范围内的值域是 $$[-1,1]$$，等于 2 这种事情想都不用想是复数。如果有背到过的话，可能会知道  $$ sin(z)=\frac{1}{2}ie^{-iz}-\frac{1}{2}ie^{iz} $$。但这个式子反正每次我用的时候都背不出来，不过我们可以通过很轻松的方法推出来，并且在这题中不需要把化简都做完反而代入计算的时候更快。

### 欧拉公式

因为欧拉公式 $$ e^{i\theta} = cos\theta + isin\theta $$，我们将所有的 $$\theta$$ 替换成 $$-\theta$$，我们有 $$e^{-i\theta}=cos(-\theta) + isin(-\theta) $$。这时候我们需要利用一点点的「奇变偶不变，符号看象限」的高中数学三角函数技巧（诱导公式），当然像我连这个都差点忘了，直接画一下函数图看一眼就会了。由诱导公式（函数的奇偶性）得到 $$ e^{-i\theta} = cos(\theta) -isin(\theta) $$，将欧拉公式和这个式子相减，我们就可以得到 $$e^{i\theta}-e^{-i\theta}=2isin(\theta)$$，把 $$2i$$ 移动一下，就得到了我们想要的关于 $$sin(z)$$ 函数的一般式子：

$$
sin(z) = \frac{1}{2i}e^{iz} - \frac{1}{2i}e^{-iz}
$$

代回原题，我们得到：

$$
\begin{aligned}
sin(x) &= 2 \\
\frac{1}{2i}e^{ix} - \frac{1}{2i}e^{-ix} &= 2 \\
e^{ix} - e^{-ix} &= 4i \\
e^{ix} - (e^{ix})^{-1} &= 4i \\
e^{ix} - \frac{1}{e^{ix}} &= 4i
\end{aligned}
$$

### 二次函数

到这里其实我们可以看到一个极其明显的 **二次函数**，我们不妨设 $$ m = e^{ix} $$，那么我们有：


$$
\begin{aligned}
m - \frac{1}{m} &= 4i \\
m^2-1 &= 4mi \\
m^2 -4im -1 &= 0 \\
m &= \frac{4i \pm \sqrt{(-4i)^2+4}}{2} \\
m &= \frac{4i \pm \sqrt{-12}}{2} \\
m &= \frac{4i \pm 2\sqrt{3}i}{2} \\
m &= 2i \pm \sqrt{3}i \\
m &= (2\pm\sqrt{3})i \\
e^{ix} &= (2\pm\sqrt{3})i
\end{aligned}
$$

### 巧妙的复数运算

$$
\begin{aligned}
e^{ix} &= (2\pm\sqrt{3})i \\
ln(e^{ix}) &= ln((2\pm\sqrt{3})i) \\
ix &= ln(i)+ln(2 \pm \sqrt{3}))
\end{aligned}
$$

这里面现在剩下最难懂的东西就是 $$ln(i)$$ 了，偷懒的人输入进 Wolfram Alpha，它告诉你答案是 $$\frac{i\pi}{2} $$。但是很可惜，这不是所有的解，我们还是最好想一下这东西是怎么来的。

我们考虑对于所有的 $$ z = re^{i\theta} $$，其中 $$ r $$ 是复数的模长，那么 $$ln(z) = ln(r) + i\theta $$。很可惜同一个 $$z$$ 不止能被一个 $$re^{i\theta}$$ 描述，因为实际上 $$ z = re^{i(\theta + 2\pi n)}, (n \in \mathbb{Z}) $$ 都成立，于是我们有 $$ln(z) = ln(r) + i\theta + 2\pi ni$$。代入 $$ln(i) = ln1 + \frac{i\pi}{2} + 2\pi ni=\frac{i\pi}{2} + 2\pi ni, (n \in \mathbb{Z}) $$。继续代回原式：

$$
\begin{aligned}
ix &= \frac{i\pi}{2} + 2\pi ni + ln(2 \pm \sqrt{3})) \\
x &= \frac{\frac{i\pi}{2} + 2\pi ni + ln(2 \pm \sqrt{3}))}{i} \\
x &= \frac{\pi}{2}+\frac{ln(2\pm\sqrt{3})}{i}+ 2\pi n \\
x &= \frac{\pi}{2}-ln(2\pm\sqrt{3})i+2\pi n, (n\in \mathbb{Z})
\end{aligned}
$$

## 阅卷过程

如果你提交的答案是：

$$x = \frac{\pi}{2} \pm ln(2 + \sqrt{3})i + 2\pi n, (n \in \mathbb{Z}) $$

这也是对的，这经过了一步非常精妙的化简，并且是大多数复分析教科书中的标准答案。

化简正确性的证明：

$$
\begin{aligned}
& -ln(2+\sqrt3) - ln(2-\sqrt3)) \\
&= -(ln(2 + \sqrt{3})+ln(2 - \sqrt{3}))\\
&= -ln((2 + \sqrt{3})(2 - \sqrt{3})) \\
&= -ln(2^2-\sqrt3^2) \\
&= -ln(4-3) \\
&= -ln1 \\
&= 0
\end{aligned}
$$


我收到了如下的错误答案：

1. $$x =  - \frac{\pi}{2}\pm i \ln\left(2 + \sqrt{3}\right) + 2\pi n, (n \in \mathbb{Z})$$。这个据提交者说是 GPT o1-preview 的答案（原版本的 LaTeX 格式甚至都有错误，我手动做了修正）。粗看差点以为是对的，仔细一看和正确答案正好差了一个 $$\pi$$。我试着上 GPT o1-preview 上复现了一下，发现算法和正确答案几乎是一样的，然后从某个时刻开始突然就想要凑教科书版本的那个答案，然后就错了...
2. $$ x=\frac{\pi}{2} - i\ln(2+\sqrt{3})+ 2\pi n  $$ 。这个答案提交者没有说是什么 AI 算的，但是我怎么看都不像人算的（原版本的 $$\LaTeX$$ 错误数量非常多，比上面那个多了一倍，我手动做了修正），而且还少了一半的解，少的位置也很离谱。

在所有声明了是 AI 做的答案中，只有一份是正确的（虽然正确的那份完全没过程，直接背了个答案上来）。我去复现了一下，发现 GPT o1-preview 算法几乎都是对的。但这题教科书版本的答案在最后有一步极其精妙的化简，当然这步不做也不影响正确性。o1-preview 一开始计算过程完全正确，但是从某个时刻开始，就拼了命地想往这个教科书版本的答案凑，然后就凑错了。甚至在凑错前 $$\LaTeX$$ 的格式至少还是对的，从凑错开始连 $$\LaTeX$$ 的格式都开始胡写了。~~人工队得分，机器队没得分。~~

## 偷鸡

把 $$sin(x) = 2$$ 输到 Wolfram Alpha 里，给了两组解：

1. $$ x = 2\pi n + \pi - sin^{-1}(2), n \in \mathbb{Z}$$
2. $$x = 2\pi n + sin^{-1}(2), n \in \mathbb{Z}$$

看起来是个基于诱导公式的废话，不过如果我们继续再在 Wolfram Alpha 里搜一下 $$ arcsin(2) $$，又得到一种 Alternate form: $$ \frac{\pi}{2}-iln(2+\sqrt{3}) $$。（我这里把 Wolfram Alpha 用的 $$log$$ 换成了 $$ln$$ 符号）。

然后代入回去得到两组解：

1. $$ x = 2\pi n +\frac{\pi}{2}+iln(2+\sqrt{3}), n \in \mathbb{Z}$$
2. $$x = 2\pi n + \frac{\pi}{2}-iln(2+\sqrt{3}) , n \in \mathbb{Z}$$

简化一下得到：

$$ x = 2\pi n + \frac{\pi}{2} \pm iln(2+\sqrt{3}) , n \in \mathbb{Z} $$

偷完了。~~传统专家模型得一分，大型语言模型没得分。~~
