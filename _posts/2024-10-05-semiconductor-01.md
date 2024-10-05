---
title: 半导体从入门到放弃（上）—— 半导体和数字电路
date: 2024-10-05 15:30:00 +0800
tags: [半导体, 数字电路, 芯片, 电路]
---

半导体芯片是现代电子设备的核心，从电视遥控器到超级计算机，无不依赖于这些微小的硅片。然而，芯片设计和制造是一个复杂的过程，涉及深厚的物理、化学知识和精密的工程技术。本篇文章旨在为读者提供一个全景式的了解，涵盖从半导体材料的基本原理到数字电路设计，以及从芯片体系结构到生产工艺的各个环节。

需要注意的是，本篇文章并不会深入探讨具体的设计方法或实现手段，除非特别举例说明。本文的目标是帮助读者理解半导体行业的核心概念，让你在面对与芯片相关的新闻或话题时，能够辨别出无良媒体的误解或错误。然而，读完这篇文章后，读者可能不会成为半导体行业的专家，甚至连入门都谈不上。但会对芯片设计背后的原理有更清晰的认识。

## 目录导航

- [半导体从入门到放弃（上）—— 半导体和数字电路](/2024/10/05/semiconductor-01)
- [半导体从入门到放弃（中）—— 局域性原理与芯片设计](/2024/10/05/semiconductor-02)
- 半导体从入门到放弃（下）—— 量产技术（未完成）

## 半导体

在讨论芯片设计的物理基础时，我们首先要了解半导体材料，尤其是硅（Si）。硅是最常用的半导体材料之一，其电子结构对于理解它的电学性质至关重要。但在理解以硅为基础材料的半导体是什么之前，我们先要对「导电」这件事情本身进行一些基础知识的理解。

### 能带理论

在单个原子中，电子只能处在特定的能量水平（称为能级），而当许多原子紧密排列形成固体材料时，这些能级会相互作用，形成连续的能量带。

我们可以将能量带想象成一座楼梯，每个台阶代表不同的能量水平。电子处在这些台阶上，它们可以在不同的台阶（能量水平）之间跳跃。当你爬得越高，你的能量就会越大（如同现实中的重力势能），反之则越小。这时候我们需要引入三个概念：

- **价带**：相当于楼梯的底部，是电子「站立」的稳定位置。这里的电子与原子核紧密结合，能量较低，移动自由度较小。电子在价带中往往是处于相对稳定的状态，无法自由移动。

- **导带**：相当于楼梯的上方，是电子可以自由「行走」的地方。电子一旦到达导带，它们就获得了足够的能量，可以在晶体中自由移动，从而形成电流。
- **带隙**：价带和导带之间有一段「空隙」，称为 **带隙**（band gap），相当于楼梯上的一大步。这一步（能量差）决定了电子从价带跃迁到导带的难易程度。材料的导电性主要取决于这个带隙的大小。

#### 导体：没有带隙的楼梯

在导体中，导带和价带是重叠的，或是非常接近，没有明显的带隙。因此，电子不需要“跨越”任何能量差即可进入导带。例如铜的电子在最外层很容易进入导带，因为它的价带和导带几乎重叠。可以想象在铜中，电子就像是在平坦的地板上行走，不需要额外的能量跃迁到导带，因而铜能够很好地导电。

#### 绝缘体：巨大带隙的楼梯

与导体不同，绝缘体（如玻璃、橡胶）具有非常大的带隙，这意味着电子要从价带跃迁到导带需要非常高的能量。在正常条件下，电子几乎不可能跨越这个巨大的能量差。例如石英的带隙非常大。电子被牢牢困在价带中，几乎无法进入导带。这就好像电子被困在楼梯的底部，无法跃过楼梯的高台阶，导致它无法传导电流。

#### 半导体：微妙的中间状态

半导体位于导体和绝缘体之间，其带隙较小，因此电子在某些条件下能够跃迁到导带，产生导电现象。硅（Si）作为一种典型的半导体材料，它的价带和导带之间有一个适中的能隙，这个能隙的大小允许我们通过外部手段（如加热或施加电场）让电子越过这个「台阶」进入导带。

硅的带隙约为 1.12 电子伏特（eV），比导体的重叠带大，但比绝缘体的小。通过适当的能量输入（如施加电场或光照），硅中的电子能够从价带跃迁到导带。这使得硅在不同条件下既可以作为绝缘体使用，也可以作为导体使用，这正是半导体材料的独特之处。

### 掺杂

硅原子在元素周期表中位于第十四族，其最外层有 4 个价电子。这些电子通过 **共价键** 与邻近的其他 4 个硅原子形成强力的化学键。硅的晶体结构被称为 **钻石结构**，是一种立体网状结构，在这个结构中，每个硅原子与周围 4 个硅原子通过共价键相连，没有任何自由电子，形成非常稳定的结构。

![钻石结构](/assets/images/silicon-structure.png)

在纯净的硅晶体中，这些共价键牢牢束缚着硅原子的价电子，使它们无法自由移动，因此在低温或常温下，硅的导电性较弱。但硅可以通过 **掺杂（doping）**，即引入微量的杂质原子，来改变它的电学性质。掺杂可以显著影响硅的 **能带结构**，从而使其导电性能发生变化。

当硅晶体中引入 **五价元素**（如磷、砷）时，这些原子带有 5 个价电子，比硅的 4 个价电子多一个。这个多余的电子并不参与共价键的形成，因此成为一个**自由电子**，可以很容易地跃迁到导带中。由于多了一个自由电子，这样的掺杂后的半导体被称为 **「N 型半导体」**。

当硅晶体中引入 **三价元素**（如硼、铝）时，这些原子带有 3 个价电子，比硅原子少 1 个，因此它们无法与周围的四个硅原子形成完整的共价键，结果在晶体结构中留下一个 **空穴**（hole）。这种掺杂产生了正电荷的载流子，即空穴，因此称为 **「P 型半导体」**。

### 二极管

![PN Junction](/assets/images/pn-junction.png)

二极管由 **P 型半导体** 和 **N 型半导体** 相互结合形成，这个接合区域被称为 **PN 结** 。在 P 型区域，主要载流子是 **空穴**（带正电的缺少电子的状态），而在 N 型区域，主要载流子是 **自由电子**（带负电的电子）。这两种材料结合在一起时，会在它们的交界处形成独特的电荷分布，进而影响二极管的导电行为。

当对二极管施加 **正向** 电压时，自由电子能够从 N 型半导体流动到缺少电子的 P 型半导体上，从而形成回路。 然而当给二极管施加 **反向** 电压时，P 型半导体少掉的电子和 N 型半导体多出来的自由电子极大阻止了自由电子从 P 型半导体向 N 型半导体传播，二极管表现为绝缘体，阻止电流通过。

### MOSFET

在理解了二极管的基本原理后，我们需要理解数字电路中非常重要的一种半导体器件 MOS 管（MOSFET）。

![MOSFET](/assets/images/mosfet.jpg)

MOS 管由三个关键部分组成：源极（Source）、漏极（Drain）、栅极（Gate）。

当栅极没有施加电压时，源极和漏极之间 **没有形成导电通道**。这意味着即使在源极和漏极之间施加电压，电流也无法通过。

我们以 NMOS 为例（图左），当在 **栅极施加电压** 时，栅极下方的 P 型衬底中的空穴被正电场排斥，电子被吸引到栅极下方，这使得电子组成了两个 N 型半导体之间的通道，使得源极和漏极之间可以被导通。PMOS（图右）则是相反，它需要通过给栅极施加负电压，排斥 N 型衬底的电子，使得源极和漏极导通。

![CMOS](/assets/images/cmos.png)

CMOS 是芯片设计中最基础的单元，它由一个 NMOS 和一个 PMOS 共同组成。在 CMOS 中，当输入信号为高电平时，NMOS 导通，而 PMOS 关闭；当输入信号为低电平时，PMOS 导通，而 NMOS 关闭。

在单独的 NMOS / PMOS 电路中，电路会有持续的电流流过，导致较高的静态功耗。相比之下，CMOS 电路在稳态时几乎没有静态功耗。在一个典型的 CMOS 电路中，NMOS 和 PMOS 交替工作，只有在状态切换时才会有电容充放电的功耗。这意味着在无切换时，功耗接近为零。这使得 CMOS 成为了现代芯片设计中最基础的单元。

我们会发现由 CMOS 管构成的电路相当于一种 **「电子的开关」**，我们可以通过控制栅极来控制电路是不是导通。但是读到这里的你可能还是非常困惑，为什么我们要使用这么复杂的方式来控制电路的开关？现代芯片是如何使用 CMOS 的组合实现各种复杂的计算、存储以及各种工作的？于是我们需要讨论 **「数字电路」** 这个概念。

## 数字电路基础

**数字电路** 的核心思想是将电流的流动（通/断）映射为 **二进制** 系统中的 1 和 0 这两种状态。这种映射让我们可以通过简单的电路组合，处理复杂的数据计算和逻辑操作。

### 基础门电路

我们不妨试着用 CMOS 搭建一下最基础的几种门电路：非门、与门和或门。

![Not Gate Off](/assets/images/not-gate-off.png)

![Note Gate On](/assets/images/not-gate-on.png)

在上面这个电路图中，当左侧给定一个高电平的信号时，PMOS 管截止，NMOS 管导通，右侧输出低电平；当左侧给定一个低电平的信号时，PMOS 管导通，NMOS 管截止，右侧输出高电平。从而我们实现了一个将 **信号反转** 的功能，这样的逻辑电路被称为 **「非门（NOT）」**。可以点击[这个链接](https://falstad.com/circuit/circuitjs.html?ctz=CQAgjCAMB0l3BWcMBMcUHYMGZIA4UA2ATmIxAUgpABZsKBTAWjDACgxCURtC8QUKGrWzdBw7gBMGAMwCGAVwA2AFyZKGk8FB0xI7AOYixQ4wISFdbAO480A0737ioNu1UzdRH05DffwDEsAlz9bTwE8ZyjI-jCBDBMJRNjXcIsHZKS0hO5ODxTQtxQY-NzwQip4tGdTMGJsvwBnAXxM80tQ7XklJoY2ACVwBvayrqoaKiQqqGgEf3tcYQCA+ICwEvdwGmE-GVSy9cqeejA52cgUNn2S6P4QmNFwc70rofXNgOJLGcnqGZg8yMK0Wi14ViAA)在线看一下非门的实现。

![And Gate](/assets/images/and-gate.png)

![Or Gate](/assets/images/or-gate.png)

类似地，我们还可以得到[与门](https://falstad.com/circuit/circuitjs.html?ctz=CQAgjCAMB0l3BWcMBMcUHYMGZIA4UA2ATmIxAUgpABZsKBTAWjDACgxCULCqwNCILjXACQ3ACYMAZgEMArgBsALk0UMJ4KNpiR2AcyEoR-QcPH4dbAO6jBKY+F7jHkNgGc7LkQ5+utcoruDGwASk5UvuAoeN46tFRIVMnQCDYguCJ4VJkgxIJu0uIY3GA0IrllFfRgqVCwKGEZkFk5LbSxyQnUXTBp4ZyRJdGxmNxdNInxfR5eY3HzfCCBwemmxaXOi2xF2Ail5Rl4JofYNXW6jba5+c0mMVDpN4LYx3kFT2+3r-edOxn7CyxH5AjKlC4Nf57bjzEGw8FJSG2FAIezDFGCKqPa7YTGnQFYtw4+yOM72SxEjK40HQ0GUsk06mEp7tQZ3Lz01licqxdb0t5snkRbG0Gi85xCvmfEzEbiS2Ui+VysUc6XiVFHPwiNyGOF4YFvbBoKzIjVY+bM5GOKIMqKUqLzDEbEVO2GA7aGBBgGHDL0w-VWcJ+kDZCjejrxSY9HSpf5iyJR4PGHII+qQRpFePgcPB1jcM7gCHp9LB0PB26UpMapPDSlZvO0QgmcN1pvZuVtqW2SXcztiVsmXs+KMD8RRrPJkW5h65-sl8NRWcfWxLnh8OdAA)和[或门](https://falstad.com/circuit/circuitjs.html?ctz=CQAgjCAMB0l3BWcMBMcUHYMGZIA4UA2ATmIxAUgpABZsKBTAWjDACgxCUKU8QUaVBMW4Cq3ACYMAZgEMArgBsALk0UMJ4KNpiR2Acwoj+go6IzcqkNgHdwGQuBo1whKmGdQ2AZ3uOPLigCTi7uIHKK3gxsAEqu7p5gvCE6tEKpMAix8eDGSXxgxlZp1MWZPn65osGFlloRUbaVtTkt1oY0eFTY2I6d3Qh11nH9KaOsQyVIZdBZ0rSdILguo8tL3GCzULAobPPOBQ60XZU94Fu6u3bjnuNuXtcn2GjH3b0PC4d9z5XW1z9gI50USmP60H5icHcNZg4FLSArRYwprjYyjSGwk68Pjo0EorGmQZ8DF7JZ4FwBWicFJnTbTHak7DkqpU-zGWkXBnXamcKg0HlHazzTCiPB8NbY9bnemQXbzHqOSW9YliqV07aytiGZXrbqEcXvKxNNYmhG6j4ml4655GuzWs3Wl5goIUzwoBD+TzOj005mU72ewI+yUB-iqkVhvjOmrsl5tY1x2PiVVgm2RpbvEPGv1JlnDVkgE78vpR7SEjKzFHUos85Kw6loHENvF2s0RiUt+GBUzFky2pb6-gWAsR+uK4e9kncxU9iF4oA)，有了与、或、非门，我们就可以围绕这三者构造大量的计算。我们不妨来看一下，我们如何使用这三个的组合来实现加法运算。

### 加法器

![Half Adder](/assets/images/half-adder.png)

如上图的[电路](https://falstad.com/circuit/circuitjs.html?ctz=CQAgjCAMB0l3BWcMBMcUHYMGZIA4UA2ATmIxAUgpABZsKBTAWjDACgAZcQqtPcMChB8o4EADMAhgBsAzg2qQ2YSiFw1uVbAiFgeooVQSdNwhIQFCU50VSlyFSJQHdTIvVTA0NL0143q4N5QbK4eZhYi1ha+UTbuGDGhwvjgiWqQGmDpvtq6+jR6pkoAshT64Tz84bzQxgAe5RBeVIRgxEH0GigaAMKSAE4DAJ7KOhndPWrjKFO81MZh+iKBIrGp0cI0vDax2xFbvPt7R1qZhyEqVDR4vFM6-LM+wgvJhRbh7yn8uTNTeRdflZ9t5Hsc2ABJWhFEQ0QjdVJUGBIRa0eHfWi3DG+UEXG6nEKuB7Ce7LOZsRoIBAdNAWQjYCBoDrdDQAeQArgAXAAOXLeRXC5k8+lK5TuGiqJOeKDqbCAA)通过逻辑门的组合，实现了两个一位二进制数的相加，被称为一个**「半加法器」**。可以想象，

![Full Adder](/assets/images/full-adder.png)

如果我们将半加法器串接，就可以处理两个二进制数和一个上一位计算是否要进位的加法器。这样的[电路](https://falstad.com/circuit/circuitjs.html?ctz=CQAgjCAMB0l3BWcMBMcUHYMGZIA4UA2ATmIxAUgpABZsKBTAWjDACgAZcQqtPcMChB8o4EADMAhgBsAzg2qQ2YSiFw1uVbAiFgeooUgSdNwhIQFCU50RClyFSJQHdTIvVTA0NL0143q4N5QbK4eZhYi1ha+UTbuGDGhwvjgiWqQGmDpvtq6+jR6pkoqQoEoKAE6wpUG1MZh+iLlqbGp0cI0vDaxXRGdvH29g1qZAyEqVDR4vLU6-BU+wqINtEXhhZGtyXk1VVZDO9WVU9PjSgCSa1v8NIQaIlQwRsl3D6nTvNuu3gt9n+dkvM9hQmrUlABZZB4fiCWFgbAWOEGaDGAAe0PIrE8KmIAnoDw0AHkAK4AFwADuTkmBIDZkbSYeAUPxfGQkcFaal-CFXKRdIJkNzBZcQMROYL+ZZRM8KMkYZ5guygj55TNmfwFSreWLgvddVkRcpVFzYSzkPTzbwVslErDgnhsLpgr47aY3TzXYj3YRYfpfL74UJHQKhCVqiGNSAtcjrVRVlr9eKNPqvRZ1ZH1Wn3d6s8lA7RNU7CzqC6whAWzuGqJH9YmlmG5Vwy4L7jjG1R7PJFGwMToa0hCLho7dOiAOJJZGSADqyADCkgATouAJ5AsEaHCRcG272LEBb4rGoTEfQsJFwfjnuovPl6jTJmXJR9UU9PHWbUyfpjF3zf4tviAP5hjSl5Aes+BXkUSgYoyeJMH0rA0OQP7kISIALsua5QoyUFIgiFjXrwqJsEAA)被称为**「全加法器」**。我们只需要堆叠这样的电路就可以实现 **任意位数的整数加法**。

![Von Neumann Architecture](/assets/images/von-neumann-architecture.png)

使用类似的技术，你可以实现各种计算、分支判断、控制等电路。然而这还缺少一个非常关键的部分，对于冯诺依曼架构而言，我们还没有解决**「存储」**这个问题。例如我们想要先算加法，再算乘法之类的，我们总是希望程序能自动执行这些步骤，并且存储中间结果，而不是我们认为每次都重新输入。

### 锁存器

![Latch](/assets/images/latch.png)

在这个[电路](https://falstad.com/circuit/circuitjs.html?ctz=CQAgjCAMB0l3BWcMBMcUHYMGZIA4UA2ATmIxAUgpABZsKBTAWjDACgxKRs8apa8VISBT8EHLjz5o8AqjP6iq4zvUL4BIDJD41Z86ioRqNYdVp3hzB5WwDu3BKJQo+Uka6hsASo9HFCbjQQAPARZHloJBQo-hhxB2wnEMCk0TMvROSM9wzIez8UwrBiJQK08FLuXkqyhz1hdX1MkCb+Nuw6uRENNoV8hz7e0wxAgYs+Myo2wj5xtrAaPm1dfTYAWQmrKgwaqcUotgAZQpc3SzO4kAAzAEMAGwBnBmp8k-cWVMtPq7unl6Q+QAHtxWCAmDxqrIIUhpHwAMoAFwA9gAnBgAHUe7BB2DBRHIPFkmCocJASLRmMe+SAA)中，你可以通过在 **时钟高电平** 时，设置 0 或者 1，使得电路保存下来你的状态。即使之后设置 0 或 1 的信号不在了，也不影响输出。这是我们第一次引入**「时钟」**这个概念，并且只让这个电路在时钟高电平时（也就是时钟上升沿执行后面的逻辑），因为在后面的电路中，我们将 **逻辑门的输出接在了逻辑门的输入** 上，这意味着本质上是**「上个时刻的输出是下个时刻的输入」**，而这个步骤如果执行地过快，可能会由于电路本身的延迟（我们会在之后很具体地讨论这个问题）导致来不及存储，从而导致程序执行的错误。为了避免这个问题，我们引入了时钟。

这种时钟边沿触发机制和时钟速率的管理，是现代芯片设计中实现稳定、可靠、高速运算的关键。如果没有时钟信号的精确控制，复杂电路的同步工作将变得难以保障，而时序错误会导致不可预测的计算结果。

### 时钟电路

#### 晶振

石英晶体（晶振）是一种压电材料，当对其施加电压时，石英晶体会发生形变，反过来，它也能够在外力作用下产生电压。当晶体振荡器工作时，石英晶体能够在特定的自然频率下发生机械振荡，称为 **谐振**。这意味着当我们将一个噪声信号传给晶振时，其谐振频率附近的频率会被放大和保留，而其它频率会被抑制。在这些噪声源的作用下，晶振可以获得启动所需的初始扰动，进而利用自身的谐振特性生成稳定的时钟信号。

#### 锁相环 (PLL)

![PLL 基本原理](/assets/images/pll-circuit.png)

**晶振** 提供的频率通常较低，无法直接满足现代处理器和其他高速电路的需求。因此，设计中常常采用 **PLL 电路**（Phase-Locked Loop，锁相环）对晶振的频率进行放大，从而得到更高的工作频率。

一个典型的PLL电路由以下几个主要模块组成：

1. **相位检测器（Phase Detector，PD）**：检测输入参考信号（例如来自晶振）的相位与PLL反馈信号的相位之间的差异，并输出一个代表相位误差的信号。
2. **环路滤波器（Loop Filter，LF）**：对相位检测器输出的误差信号进行滤波，去除高频噪声，并将其转化为一个相对平滑的控制电压信号。
3. **压控振荡器（Voltage-Controlled Oscillator，VCO）**：将滤波器输出的控制电压转换为相应频率的振荡信号。VCO的输出频率随着控制电压的变化而变化。
4. **分频器（Frequency Divider，FD）**：将VCO输出的高频信号进行分频，生成与输入参考信号相近的低频信号，作为反馈信号输入给相位检测器。

PLL 的核心在于**相位检测器**，它对输入的参考信号（晶振提供的频率较低的信号）与**反馈信号**（来自 VCO 输出的信号经过分频器处理后的信号）进行比较，输出一个与相位差成正比的信号。这个相位差会引发控制电压的变化，进而调节 VCO 的输出频率。

- 如果 VCO 输出的频率过低，控制电压会升高，从而使 VCO 频率增加；
- 如果 VCO 输出的频率过高，控制电压会降低，降低 VCO 的输出频率。

**分频器** 用于将 VCO 输出的高频信号按比例降低，使其与参考信号的频率接近。分频器通常会将 VCO 的输出频率除以一个特定的整数值（N），然后将这一频率反馈给相位检测器。这使得 PLL 电路可以通过控制 VCO 的高频输出与低频的参考信号进行比对，从而调节 VCO 的输出频率。例如晶振的频率是 10MHz，而我们希望得到 100MHz 的输出时钟信号。通过将 VCO 输出的 100MHz 信号分频 10 倍，得到一个 10MHz 的信号，然后与晶振的 10MHz 信号进行相位比较。这种设计使得 VCO 的输出频率被锁定在晶振频率的整数倍上。

现代 PLL 电路中，通常会采用一些更加复杂的技术来实现更精确的频率控制和调节。这包括分数倍频和多级 PLL 串接。这些技术可以生成更加灵活、精确的时钟信号，但它们的原理和上文其实没有本质区别，因此在本文中我们不深入讨论它们。

### 小结

![MOS 6502](/assets/images/mos6502.jpeg)

结合上面的知识，你已经几乎可以设计出类似于 1971 年的 Intel 4004 或者 1975 年的 MOS 6502 这样的经典芯片了。不过这样的芯片性能非常慢。在[下个章节](/2024/10/05/semiconductor-02)我们会讨论如何通过改进芯片设计来改善运行性能。
