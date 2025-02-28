---
title: 当我们吹嘘量子并行计算的时候
date: 2025-02-28 13:00:00 +0800
tags: [量子计算, 量子物理, 物理]
---

昨天晚上和老金聊到微软的 Majorana 1 的时候，我说了一段关于量子计算研究的评论。我觉得很有必要再说一次。

我想起 2016 年加拿大总理特鲁多给记者解释量子计算的视频在互联网上被疯传[1]。特鲁多对量子计算的解释概括来说是基于「量子并行计算」概念的，也就是说传统的比特只能同时表达 0 或 1，随着位数的增加，为了在一个庞大的搜索空间中寻找到答案，我们需要枚举所有的答案；而量子比特（qubit）可以同时表达 0 和 1，因此一组 qubits 可以同时表达所有可能的答案，并行地找到我们想要的答案。

我对「量子并行计算」概念的评价就是「正确的废话」，类似的诠释出现在许多科普读物、维基百科等地方。我认为当特鲁多背诵这段内容的时候，他必然是对量子计算一无所知的。

我们不妨考虑一个问题。如果一组量子比特（qubits）在同时表达 0 和 1，那么我们如何「找到我们『想要的』答案」呢？这个「想要」事实上是相当关键的。因为不管量子比特如何表达，基于玻恩公设（Born rule）[[2]][[3]]，你最终只能通过测量（measurement）得到其中一个结果，这个结果还是 0 或者 1 的，其概率为量子太处于本征态的概率幅绝对值的平方。如果量子是单纯地同时表达了所有可能的答案，那么测量的结果就是在状态空间下的随机数发生器，那不叫量子计算，那个叫[量子算命](/2019/11/13/quantum-poe)[[4]]。

秀儿算法（Shor's Algorithm）[[5]]向我们展示了量子计算不是算命的关键差异。其通过引入一部分量子算法，使得其可以更快地寻找到余数循环的周期，从而帮助进行大数的因数分解。其中这个寻找余数周期的方法，被称为量子傅里叶变换（Quantum Fourier Transform, QFT）。这其中最巧妙的部分不是把量子比特看成同时表达 0 和 1 的数字，而是看成一个波函数。通过构造波和波之间巧妙的干涉作用，相干相消后使得我们「想要的答案」在其本征态的概率幅大幅上升，而其它答案大幅下降。基于这样的前提，我们才能通过测量其量子态，得到某个我们想要的答案。

这个算法充分体现了量子计算和传统计算机计算方式的根本不同，它们之间以目前的情况来看，不存在简单的替换关系。把量子计算理解成飞跃式的并行计算机是没有道理的。更何况今天量子计算机在工程上仍有大量东西值得解决。

像 Majorana 1 试图构造凝聚态物理上等效的 Majorana 费米子[[6]]，从而使得其具有拓扑的 qubit 性质，我个人的感觉是在量子退相干问题上可能会有更好的稳定性。首先我不太懂凝聚态，其次我不太懂拓扑，但从论文上看，好像论文作者也不太确定这是不是 Majorana 费米子。但这其实是量子计算在学界上一个很小的突破，和 PR 稿上的大鸣大放可以说是没有任何关系了。

当然以今天量子计算的进展，还是得编故事的。算了，你总不能指望投资人真的理解啥是什么量子态本征态，算个 sin 函数还带复数的，最后还要扔进什么恐怖积分式子里积半天的，已经超越商科数学只能算加减乘除的界限了。不如就编这种东西吧，不编咋骗人投钱来做科研，不做科研未来人类计算怎么进步。

References:

1. Reuters. (2016, April 17). Internet abuzz after quantum computing lesson by Canada’s Trudeau. Reuters.
2. Claude Cohen-Tannoudji, Bernard Diu, Franck Laloë. Quantum Mechanics Volume 1. Hermann.
3.  Born, Max (1926). "Zur Quantenmechanik der Stoßvorgänge" [On the quantum mechanics of collisions]. Zeitschrift für Physik. 37 (12): 863–867.
4. Ding, D. (2022). dsh0416/quantum-i-ching. https://github.com/dsh0416/quantum-i-ching
5. Shor, Peter W., Polynomial-Time Algorithms for Prime Factorization and Discrete Logarithms on a Quantum Computer, SIAM J.Sci.Statist.Comput., 1999, 41 (2): 303–332, doi:10.1137/S0036144598347011.
6. Aghaee, M., Alcaraz Ramirez, A., Alam, Z., Ali, R., Andrzejczuk, M., Antipov, A., Astafev, M., Barzegar, A., Bauer, B., Becker, J., Bhaskar, U. K., Bocharov, A., Boddapati, S., Bohn, D., Bommer, J., Bourdet, L., Bousquet, A., Boutin, S., … Zilke, J. (2025). Interferometric single-shot parity measurement in InAs–Al hybrid devices. In Nature (Vol. 638, Issue 8051, pp. 651–655). Springer Science and Business Media LLC. https://doi.org/10.1038/s41586-024-08445-2
