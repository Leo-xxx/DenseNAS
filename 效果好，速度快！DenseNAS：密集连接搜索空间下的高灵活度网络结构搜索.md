## 效果好，速度快！DenseNAS：密集连接搜索空间下的高灵活度网络结构搜索

[我爱计算机视觉](javascript:void(0);) *昨天*

文章转载自公众号 ![地平线HorizonRobotics](http://wx.qlogo.cn/mmhead/Q3auHgzwzM6flk4shtstX88WYASHMzPPupnPyCGPmuibL1iaTz32VCWA/0) 地平线HorizonRobotics ， 作者 平台与技术部

![img](https://mmbiz.qpic.cn/mmbiz_jpg/KsD6E81MroCL6CkygEImsicv4wibkvVa5BzfIKWb3pm4kmC7Sg5cQwf5bG9wNj7cU7R0aia4xN3yQ8UQs9BibgBlNg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

近年来，网络结构搜索（NAS）在自动化设计神经网络结构方面获得了较大的成功，也成为模型结构优化领域不可忽视的重要研究课题。NAS 不仅减轻了人们设计、调优模型结构的重重负担，而且相较于人工设计的网络结构，搜索出的模型性能有了进一步提升。

最近，地平线-华中科技大学计算机视觉联合实验室提出了一个新颖的 Differentiable NAS 方法——DenseNAS， 该方法可以搜索网络结构中每个 block 的宽度和对应的空间分辨率。本文将会从简介、对于网络规模搜索的思路、实现方法以及实验结果等方面诠释 DenseNAS 这一新的网络结构搜索方法。

论文地址：https://arxiv.org/abs/1906.09607

代码地址：https://github.com/JaminFong/DenseNAS



![img](https://mmbiz.qpic.cn/mmbiz_png/KsD6E81MroCL6CkygEImsicv4wibkvVa5BLCzFWNEztq5HiaTpOgIK7UF1CrsGdwjZNIhKicoZQzmco4OBoDsmVFFw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)









**DenseNAS 简介**



NAS 极大推进了神经网络结构设计的发展，然而很多以往的工作依然需要很大的计算代价。最近 Differentiable NAS 通过在连续空间上构建一个包含所有要搜索结构的搜索空间（super network）极大的减少了搜索代价，但事实上，很少有Differentiable的方法可以搜索网络结构的宽度（即通道数），因为按照传统Differentiable NAS的方法，将不同宽度的结构集成到一个 super network 里面很难实现。在本论文中，我们提出了一个新颖的 DifferentiableNAS 的方法——DenseNAS，该方法可以搜索网络结构中每个 block 的宽度和对应的空间分辨率。我们通过构建一个密集连接的搜索空间来实现该目的。在我们设计的搜索空间中，拥有不同宽度和空间分辨率的block之间相互连接，搜索过程中优化block之间的转移概率从而选取一个最优路径。DenseNAS 使得网络结构搜索的灵活性更强，以宽度搜索为出发点，同时可以搜索网络结构下采样的位置和全局深度（不仅限于每个block中的层数，block的数量也会被搜索）。**在 ImageNet 上，DenseNAS 得到的模型以较低的 latency 取得了 75.9% 的精度，整个搜索过程在 4 块 GPU 上仅用了 23 个小时。**

DenseNAS 以其更高的灵活性应用潜力也更大，可以用于特定场景数据的结构搜索、特定性能和速度需求的搜索以及特定设备的结构部署，因为其在搜索空间上的弹性更大，也可以用于对 scale 敏感的方向，如检测、分割等任务。









**NAS搜索元素的梳理**



设计神经网络结构是深度学习中非常重要的一个领域，近年来 NAS 在自动设计神经网络方面取得了很大的成功。很多 NAS 方法产生的模型结构与人工设计的结构相比都体现出更优的性能。目前在诸如分类、分割和检测等各方向 NAS 均有进展。NAS 的方法不仅能够提升模型的性能，另一方面还能够减轻人们设计、调优模型结构的负担。

模型结构设计过程中可以搜索的元素越多，相应工程师的负担就越小。哪些元素能够被搜索又取决于搜索空间如何设计。在以往的工作中，操作（operation）类型的搜素已经实现的较好，但是搜索网络的规模（宽度和深度）就没有那么直接。基于增强学习（RL）或者进化算法（EA）的 NAS 方法能够比较容易的搜索宽度、深度，因为他们的搜索空间在一个离散的空间中，但是这类方法往往需要消耗非常大的计算代价。最近 Differential 和 one-shot 的方法能够用极少的搜索代价来取得高性能的网络结构，但是网络规模的搜索却不太容易处理。这类方法的搜索依赖于一个包含所有可能结构的超大网络（super network），网络规模的搜索需要将不同规模的结构全部整合到 super network 里面。目前深度搜索通过在每一层的候选项里面增加直连（identity connection）操作来实现，但宽度的搜索依然不容易处理。









**DenseNAS对于网络规模搜索的思路**



深度和宽度的设定往往对结构性能产生很大的影响，特别是通常微小的宽度改变都可能造成模型大小爆炸式的增长，现在的搜索方法中宽度通常由人提前设定好，这需要模型结构方面专家很强的经验。我们旨在解决基于 Differentiable NAS 的宽度搜索问题，从而提出了 DenseNAS 的方法。我们的方法构建了一个密集连接的搜索空间，并将搜索空间映射到连续可操作的空间。不同于 DenseNet，我们的搜索过程会选择一条最佳的宽度增长路径，最终只有一部分 block 会被选中并且最终结构中的 block 之间不会再有连接。在搜索空间中每个 block 对应不同的宽度和空间分辨率，从而不仅宽度会被搜索，进行下采样的位置和全局的深度（block内层数+block的数量）都会被搜索，这使得整个搜索的过程更加灵活。

![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)









**方法介绍**



## 1.密集连接搜索空间的构建

我们将整个搜索空间划分为几个层次：层（layer）、块（block）、网络（network）。

每个 layer 包含各种操作候选项，候选操作基于 MBConv，同时也包含 skip connection 用来做深度搜索。

每个 block 由层组成，一个 block 被划分为两个部分头层（head layers）和堆叠层（stacking layers）。我们对每个 block 设定一个宽度和对应的空间分辨率。对于头层来说，其输入来源于前几个 block 不同通道数和空间分辨率的数据。头层是并行的，将所有输入数据转换到相同通道数和空间分辨率；堆叠层是串行的，每层被设定在相同通道数和分辨率下，并且每层的操作可搜索。

不同于以往的工作，block 的数量固定，我们的搜索空间包含更多不同宽度的 block ，最终只有一部分被选取，这使得搜索的自由度更大。整个 network 包含几个 stage，每个 stage 对应一个范围的宽度和固定的空间分辨率。网络中 block 的宽度从头到尾逐步增长，每个 block 都会连向其后继的几个 block。

![img](https://mmbiz.qpic.cn/mmbiz_png/KsD6E81MroCL6CkygEImsicv4wibkvVa5BoKykIFQbj0mepeduGGHjicQ3GjmEA7GV0NNgy1ia6rSywD6S5O5qIo1A/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## 2.搜索空间的连续性松弛

对于 layer 层次，每个候选操作被赋予一个结构参数，layer 的输出由所有候选操作输出的加权和得到：

![img](https://mmbiz.qpic.cn/mmbiz_png/KsD6E81MroCL6CkygEImsicv4wibkvVa5BfGyduia4UpSLwDGVHAKGGCj8iaZ3ceribv5h7zIoYiaibPeCEC2eEl1tKxQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

对于 block 层次，每个 block 的数据会输出到其后继的几个 block，每条输出的路径同样会被赋予一个结构参数，并通过 softmax 归一化为输出概率。每个 block 会接受前继几个 block 的输出数据，在 head layers 部分，会对来自不同 block 的数据利用路径的概率值进行加权求和：

![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

## 3.搜索算法

整个搜索过程被分为两个阶段，第一阶段只有 operation 的权重参数被优化；第二阶段 operation 的权重参数和结构参数按照 epoch 交替优化。当整个搜索过程结束后，我们利用结构参数来导出最终的结构。每一层的操作将选择结构权重最大的候选项；在 network 层面，我们利用 Viterbi 算法来选择整个传输概率最大的路径，仅有一部分 block 被选中。

搜索过程中我们加入了多目标优化，latency 被作为次优化目标，通过查表的方法被优化。

![img](https://mmbiz.qpic.cn/mmbiz_png/KsD6E81MroCL6CkygEImsicv4wibkvVa5BlicY7WgbtIIVNLm8JSG78m97AekvkiaecPA2MbDscF4m7fw83X9iaJ4bg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

参数优化过程中我们通过概率采样 path 来进行加速。对于 operation 的权重参数，采样 path 的优化方法不仅可以起到加速、减少显存消耗的作用，还可以在一定程度上降低不同结构 operation 之间的耦合效应。









**实验结果**



DenseNAS 在 ImageNet 上搜索的结果如下表所示。我们设置 GPU 上的 latency 为次优化目标，DenseNAS 搜索得到的模型在低 latency 下取得了优良的精度。在同等latency的设定下，DenseNAS 的精度远高于人工设计的 MobileNet 模型。和 NASNet-A、AmoebaNet-A 和 DARTS 等经典的 NAS 模型相比，DenseNAS 模型精度更高，FLOPs 和 latency 更小。在搜索时间上 DenseNAS 仅在 4 块 GPU 上花费 23 小时（92 GPU hours）。和 Proxyless、FBNet 相比，我们的宽度均为自动搜索，并取得了卓越的模型性能。

![img](https://mmbiz.qpic.cn/mmbiz_png/KsD6E81MroCL6CkygEImsicv4wibkvVa5Blg1TTmg0XPicEpYMvxMbfQQfw8BUYiahZy0nDe76nZ5Pib56icKyYB9sAw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

DenseNAS 进一步在不同程度 latency 优化下搜索模型，在各个 latency 设定和需求下，都能得到性能优越的模型结构，均要远好于固定宽度/block 搜索和人工设计的模型。

DenseNAS 搜索得到的模型结构如下图所示：

![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)



论文地址：https://arxiv.org/abs/1906.09607



代码地址：https://github.com/JaminFong/DenseNAS

![img](https://mmbiz.qpic.cn/mmbiz_png/KsD6E81MroCL6CkygEImsicv4wibkvVa5BnCcTtKib7HDYm9sEw1400M6YoDuEawU23BNqiaWXA63M5YcgVRia9vlHw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

关于作者

方杰民 & 孙玉柱，地平线平台与技术部算法研发二部算法实习生（mentor：张骞 & 李源），主要研究方向为网络结构搜索、模型结构优化。该项目完成于他们实习期间。他们于华中科技大学电子信息与通信学院人工智能研究所研究生在读，师从刘文予教授和王兴刚副教授。

**NAS神经架构搜索交流群**



NAS已成为被各个大厂竞相追逐的人工智能新方向，关注最新的NAS技术，欢迎扫码添加CV君拉你入群，

**（请务必注明：NAS）**

**![img](https://mmbiz.qpic.cn/mmbiz_png/BJbRvwibeSTs1Ke4WXicIqN7QibMXL527MCvicgajlnePVw1mnomoLqFqL0WLf7UUpSkVGj2E1GGe83e8ZmY0G42jw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)**

喜欢在QQ交流的童鞋，可以加52CV官方**QQ群**：805388940。

（不会时时在线，如果没能及时通过验证还请见谅）

![img](https://mmbiz.qpic.cn/mmbiz_png/BJbRvwibeSTvVOnJBvePcP1qFUSWpyvrjpYAWNIZTZzUA7Zq4VPlReicJWcIeozxic5VhHlwNQNAFXmKQBtKf5xAQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

长按关注我爱计算机视觉











微信扫一扫
关注该公众号