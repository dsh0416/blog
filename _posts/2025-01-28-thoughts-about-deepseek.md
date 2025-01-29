---
title: 写在甲辰年末
date: 2025-01-28 23:00:00 +0800
tags: [AI]
---

今天有朋友问我：「你居然没点评 DeepSeek，不习惯。」
好吧，看来大家还是喜欢看我大放厥词。

Deepseek R1 的论文[1](https://doi.org/10.48550/ARXIV.2501.12948)我上周就已经看了，当时是很震撼的。我看到那个 rule-based RL 的时候，真得有种 Aha 的感觉。一些人可能知道，受一些论文[2](https://doi.org/10.48550/ARXIV.1412.7449)[3](https://doi.org/10.48550/ARXIV.2305.19234)的启发，我们在研发的系统广泛受益于 grammar-constrained sampling 下对 LLM 的调用的效果。但我也从来没有意识到过可以通过如此精巧得构造使得一个自监督的 RL 可以产生如此精妙的效果。但今天我想跳出纯学术角度来讨论一些问题。

我想我们应当确立一个基本的共识。Scaling Law[4](https://doi.org/10.48550/ARXIV.2001.08361) 和 Moore’s Law 一样，是一种结果而不是原因。科技的进步从来没有银色子弹。没有人会相信 Moore’s Law 会成立，大家不但知道晶体管密度不可能持续地翻倍下去，也知道计算机性能不可能持续地翻倍下去。但是无论是诸如 FinFET、Gate-All-Around 的半导体器件工艺的进步，或者像是 DUV、EUV、浸润式光刻等制造技术的进步，以及从微架构设计和体系结构出发的芯片设计水平的进步，是一代代人智慧和努力的结果让 Moore’s Law 看起来仍然成立。类似地，AI 水平的进步绝无可能是单纯依赖参数量、算力、数据集的进步。如果有人告诉你今天 AI 发展会逐渐停滞，因为互联网上所有数据集都被 AI 吃光了[5](https://www.economist.com/schools-brief/2024/07/23/ai-firms-will-soon-exhaust-most-of-the-internets-data)，你不觉得可疑吗？如果一个人类有能力理解互联网上那么多数据，它会像这些 AI 一样笨吗？真的不是显而易见地，我们的方法出了什么问题吗？

当然资本市场自有它自己的看法。虽然我不止一次公开说过，在我心目中 Sam Altman 就是个傻◯，但我不否认 Sam 是一位优秀的 CEO，资本市场对他的吹捧是他应得的。我认为问题的本质是政治问题。只要人类社会存在一天，就会有一天主义。归根究底是我们就无法避免对有限资源争夺的分配问题。Sam Altman 把 AI 的能力和参数量、数据集、算力关联起来，试图证明其先发优势是不可撼动的，来试图获得更多的资本或资源，以在竞争中获得优势；对应地 NVIDIA 也欣然接受这套理论，因为这意味着资本市场会相信 NVIDIA 能源源不断地卖出更多的卡。在这条路上走得最远的是我听过最离谱的故事是把 AI 能力和电力和能源关联起来。这些理论细想之下根本不值得推敲，无论是我们从定域性原理[6](https://doi.org/10.1103/revmodphys.92.021002)的角度还是兰道尔原理[7](https://doi.org/10.1016/s1355-2198(03)00039-x)的角度，我们都应该很直觉地意识到，这些关联和人类已知的其它智能系统相比，都有至少 6 个以上数量级的差距，今天我们所遇到人工智能的发展瓶颈，绝无可能是这些限制，而是方法的限制。但精通政治的人类会发现一条捷径，只需要将筹码都握在自己手中，就能避免其他人找到正确的路径。那么无论如何下一个里程碑永远是自己的。甚至这种捷径可能是对行业整体更有利的，因为这让钱流向了 AI 行业而不是其它行业。

捷径所省下的工作，最终都是要还债的。CEO 所需要欺骗的对象，是一群最多只能记住 Y=C+I+G+(X-M) 的人，想必这个捷径也很有它的走法。于是 OpenAI 变成了 ClosedAI。就像很多人考学到顶尖学府的顶尖专业，其实对那个专业的研究没有丝毫热情，只是想要毕业后顶着学历去金融公司赚份快钱，而他真正挡住的，是有机会在这个领域研究上发光发热的人，间接地破坏的是人类的利益。但我总是相信，人类历史上所有这些围绕封闭而构成垄断的虫豸未曾成功过，不单单是这样的谎言总是会被戳破，人们终会意识到「以前的蔷薇的梦原来都是虚幻，现在所见的或者才是真的人生」；更重要的是，人类自会找到问题的出路，任何形式的封闭、保密或者封锁都是不可能在长期的尺度上成立的。

最后，我想要借这个机会，讲一讲我家小猫名字的故事。我家小猫的名字叫伽罗瓦，以纪念我最喜欢的法国数学家之一，埃瓦里斯特·伽罗瓦（Évariste Galois）。没有可信的历史告诉我们伽罗瓦是如何去世的，虽然最浪漫的说法是他与人决斗而死。他对代数的前卫研究一直等到伽罗瓦去世 8 年后刘维尔（Joseph Liouville）整理伽罗瓦的遗稿时才被重视[8]。但是这并不影响伽罗瓦帮助人们终于解开了对数学研究工具的桎梏，开创了代数研究的重要新学科。

我想留给大家的话是 Stay Humble，坚持面对问题，然后像西西弗斯一样去解决它，除此之外，我们似乎也别无他法。

写在甲辰年末。

References:
[1]: DeepSeek-AI, Guo, D., Yang, D., Zhang, H., Song, J., Zhang, R., Xu, R., Zhu, Q., Ma, S., Wang, P., Bi, X., Zhang, X., Yu, X., Wu, Y., Wu, Z. F., Gou, Z., Shao, Z., Li, Z., Gao, Z., … Zhang, Z. (2025). DeepSeek-R1: Incentivizing Reasoning Capability in LLMs via Reinforcement Learning (Version 1). arXiv. https://doi.org/10.48550/ARXIV.2501.12948
[2]: Vinyals, O., Kaiser, L., Koo, T., Petrov, S., Sutskever, I., & Hinton, G. (2014). Grammar as a Foreign Language (Version 3). arXiv. https://doi.org/10.48550/ARXIV.1412.7449
[3]: Wang, B., Wang, Z., Wang, X., Cao, Y., Saurous, R. A., & Kim, Y. (2023). Grammar Prompting for Domain-Specific Language Generation with Large Language Models (Version 3). arXiv. https://doi.org/10.48550/ARXIV.2305.19234
[4]: Kaplan, J., McCandlish, S., Henighan, T., Brown, T. B., Chess, B., Child, R., Gray, S., Radford, A., Wu, J., & Amodei, D. (2020). Scaling Laws for Neural Language Models (Version 1). arXiv. https://doi.org/10.48550/ARXIV.2001.08361
[5]: The Economist Newspaper. (n.d.). AI firms will soon exhaust most of the internet’s data. The Economist. https://www.economist.com/schools-brief/2024/07/23/ai-firms-will-soon-exhaust-most-of-the-internets-data 
[6]: Wharton, K. B., & Argaman, N. (2020). Colloquium : Bell’s theorem and locally mediated reformulations of quantum mechanics. In Reviews of Modern Physics (Vol. 92, Issue 2). American Physical Society (APS). https://doi.org/10.1103/revmodphys.92.021002
[7]: Bennett, C. H. (2003). Notes on Landauer’s principle, reversible computation, and Maxwell’s Demon. In Studies in History and Philosophy of Science Part B: Studies in History and Philosophy of Modern Physics (Vol. 34, Issue 3, pp. 501–510). Elsevier BV. https://doi.org/10.1016/s1355-2198(03)00039-x
[8]: Bruno, L. C., & Baker, L. W. (1999). Math and mathematicians: The history of math discoveries around the world. U X L.
