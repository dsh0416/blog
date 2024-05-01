---
title: 量子算命，在线掷筊 —— IBM 量子云计算机使用入门
date: 2019-11-13 02:54:20 +0900
tags: [量子计算, 量子物理, 物理]
---

## 背景

自古以来算命离不开随机现象。古代人使用龟壳的裂纹来进行占卜，但出于动物保护主义，现代社会使用裂纹之类的自然混沌来预测变得不人道。现代算命会使用洗牌等手段产生随机数，但这样的随机很多时候可以得到人为控制，无法真正体现神明的意思。

作者在此提出一个方法，利用量子力学现象进行随机数的发生。我们对于一个量子位（qubit），使其通过一个阿达马门（Hadamard gate）。这个门可以将 $\vert0\rangle $ 转换成 $\frac{ \vert0\rangle + \vert1\rangle } {\sqrt{2}} $，即进行状态叠加。当我们此时测量这一量子的状态，那么其波函数会发生突变，变为其中一个本征态。且在我们所设计的量子电路中，观察到 0 和 1 的概率应该都是 50%。如果我们今天对量子力学的认识是正确的，那么随机性应该是量子物理的内禀性质，所以我们在邀请真正的上帝来为我们掷骰子，能真正表达量子神明的意愿。

## 实现

我们采用 IBM 的量子云计算机 IBM Q 进行实现。IBM 提供了丰富好用的量子程序开发的 SDK，并且可以在设计调试完成后，交给线上真正的云量子计算机进行运算，让你足不出户，不需要亲自前往土地庙，也可以听到神明的解答。

我们以「掷筊」为例，

> [掷筊](https://zh.wikipedia.org/zh-hans/%E6%93%B2%E7%AD%8A)是一种道教与民间信仰中问卜的仪式；又称掷筶、掷杯、博杯，普遍流传于华人民间传统社会。「筊杯」是一种占卜工具，是世俗之人所用以与神明指示的工具。

我们同时观测两个处于叠加态的量子。若它们都是 0，那么就是「阴筊」；如果都是 1，那么就是「笑筊」；如果两个状态相反，那么则是「圣筊」。具体的量子电路如下：

![Poe Circuit](/assets/images/poe-circuit.png)

使用 Qiskit 实现起来非常便利：

```python
# Setup Dependencies
from qiskit import *

# Setup Poe Dictionaty
POES = ['陰筊', '聖筊', '聖筊', '笑筊']

# Setup Quantum Circuit
circ = QuantumCircuit(2)
for i in range(2):
    circ.h(i)
circ.measure_all()

# Use Simulator for Testing
backend = Aer.get_backend('qasm_simulator')

# Execute the Result
job = execute(circ, backend, shots=1)
result = job.result().get_counts(circ)

# Convert results
converted = int(list(result.keys())[0], 2)
print(POES[converted])
```

我们只需要每进行一次这个量子电路的执行，就可以得到一次上天的掷筊结果。在模拟器测试成功后，我们就可以将后端换成真正的 IBM 云量子计算机进行掷筊。（完整样例程序[参考这里](https://github.com/dsh0416/quantum-i-ching/blob/master/poe.ipynb)）

## 未来工作

我们使用了这一技术更有效的让神明为我们解答问题，对于人类社会具有重大意义。再也不需要等妈祖来托梦，人类可以主动去探寻神明的秘密。但是这一方案目前仍存在不足。

例如如果我们基于易经进行六爻占卜（[程序](https://github.com/dsh0416/quantum-i-ching/blob/master/notebook.ipynb)），一共需要 64 个经典态，即需要 6 个量子。但 6 个量子的量子芯片目前仍十分稀有。但由于我们的占卜过程不需要产生任何纠缠态，只需要 H 门这一种量子门，在此我们提出可以针对这一市场需求设计并行的单量子芯片，从而让量子计算走入千家万户。
