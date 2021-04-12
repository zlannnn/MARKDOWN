# ResNet的堆叠与数值方法

细节和理论可以看文章和本文末链接的两篇文章。这里就说得稍微笼统易懂一些。

## Neural Ordinary Differential Equations [1]
NODEs是NIPS18的best paper。 核心内容是把DNN看做离散化特例，那么如果DNN的层数拓展到无限深，就自然化为了连续的形式。
NODEs把input到output的mapping过程化为一个在特定点求解ODE的初值问题，引入了ODE求解器来完成，从而实现了O(1)参数量。

![image](https://user-images.githubusercontent.com/15451867/114329774-31940900-9b7b-11eb-9e0e-21f91e8d872a.png)
在此之前其实很多研究者也有过类似的想法发表，详情请看LM-ResNet[2]的作者2prime大佬当时的一个总结介绍：https://zhuanlan.zhihu.com/p/51514687

我们不去讨论这个idea到底应该归功于谁，不论如何，NODEs都是非常精彩漂亮的一篇文章，提供了一个ODESolver library，并且极大的引发了人们对这个方向的关注。

## ODE和DNN的联系
ODE和近几年大热的DNN冥冥之中有着千丝万缕的联系。像ResNet, DenseNet，设计的时候似乎并没有考虑到数值形式，是后来被归纳进去的。有的则依据ODE的理论进行了设计，比如LM-ResNet，依据的是线性多步。随着越来越多的网络被归纳进来，就不禁让人疑惑，ODE理论是否能够为NN的设计提供宏观上的指导？
![image](https://user-images.githubusercontent.com/15451867/114330535-e24ed800-9b7c-11eb-91f7-8feca3fb2b8b.png)

在使用NODEs的求解器的时候，我们可以大概观察到一个正比的关系。在求解精度设置一致的情况下，如果求解器所使用的方法是更高阶的数值方法，那么最后NN的性能大多数情况下会有所提升。但是在NODE定参数的情况下，这样做会急剧提升浮点运算次数，从实践上看，似乎又不是那么划算。

但既然NODE是把ResNet从一阶欧拉前向总结到NODE中，然后建立solver调用高阶数值办法。那反过来，高阶数值方法也可以离散化，成为高阶的NN，实际上，很多NN也已经被归纳到了这个框架下。比如RK-4方法也在ICLR中发表过RK-Net，VoVNet的输出形势也很类似RK-4。

## Block堆叠和数值方法
长久以来，DNN的设计都以sub-net为主。VGG风格的两层3x3或者是bottleneck。这其实很好理解，ResNet突然把深度从十几层拉到1000层，人为的设计每一层确实显得不那么合理。这种从ResNet开启的2-3层乘以X就形成了XXNet(subnet,[S1,S2,S3,S4])的模型构筑风格。在最新的ResNet-RS中甚至有单个stage可以堆叠84个ResBlock。

因为前人已经做了很多Sub-net设计的工作，因此这次考虑数值方法的时候，我们的视角更多的放在了block之间的链接上。在layer之间每两层就要加一个shortcut，如果把2-3层的sub-net看作是一层layer，直接前向堆叠几十次的做法感觉上收益会迅速递减。

![image](https://user-images.githubusercontent.com/15451867/114331966-25f71100-9b80-11eb-9b5d-bf470a3fc51d.png)
有一张数值方法的图可以很好的表达为什么我们可以从高阶的堆叠方式中获取收益。

## LR与初始状态
以ResNet为例，kaiming大神提出ResNet的时候是为了解决深层不进反退问题。虽然ResNet非常简洁优雅地，极大地缓解了这个现象，并成为DL里程碑式网络，但在他的论文中结果其实也显示，1000层的ResNet还是退了。在我的实验中，不仅1000层的ResNet退了，不到100层的ResNet也出现了退化情况。这并不是说它们不能够取得比浅层更好的结果，而是它们对LR和初始状态这样的超参数非常敏感。

梯度的消失和爆炸都与链式法则密切相关，细微的差别在极深的网络中会被传播放大，而如上图所示，欧拉法对比更高阶的方法更容易有误差。在CIRAR-10上，如果你用0.1LR的SGD训练一个小于40层的网络，效果非常好。但随着深度的持续增加，你就会观察到ResNet的前期收敛明显变得慢了，在极深的情况下仍旧会出现梯度爆炸这个老问题。

这时候如果适当调节学习率，问题就会得到解决，我相信大家在各种项目中也饱受过学习率之苦，都有过类似的经验。可正因为微调一下学习率就解决了问题，这才让人很容易忽视背后的问题。我们经过试验发现，在大lr导致较深ResNet早期收敛变慢的时候，高阶的ResNet仍旧可以一切如常，快速收敛。其现象与上图所示一致。我们特意寻找到ResNet会梯度爆炸的设置，高阶堆叠的ResNet同样可以正常快速的收敛。

基于这个现象，我们对鲁棒性也进行了进一步的测试。假设一个非常简单的情况，Conv(3,64)，然后堆叠N个(64,64)维度的VGG风格双层3x3的block。整个过程中，仅有一次stride=2的feature下采样。对比两种情况：1，下采样发生在Conv(3,64)阶段，在升维的过程中，下采样。2.在第一个Conv(64,64)layer下采样。结果一个简单变化，Baseline ResNet的性能就发生了较大的波动，而高阶堆叠的ResNet则非常稳定。

## 同参数的堆叠对比
高阶方法之所以繁琐，是因为在同步长情况下进行了更多次运算，计算了更多的中间状态以获得更为精确的数值。这也是为什么前文说到，NODEs中调用求解器，高阶办法需要几倍的浮点运算次数。

而我们设计的对比实验，则是把反复堆叠的低阶办法看作是一种另类的高阶办法。以实际参数量和符点运算次数相等的情况来做公平对比，因此虽然名义上是高阶堆叠的ResNet，但由于Block的数量不同，实际上是同层数，深度，参数量和计算量的。

![image](https://user-images.githubusercontent.com/15451867/114336096-2ea01500-9b89-11eb-822b-53acdefc1abe.png)
以ResNet-50为例，如采用VGG风格的2层3x3设计。标准ResNet也就是欧拉前向法，就应当是堆叠24次。Res-50(2+2x24)。若采用二阶办法，则只堆叠12次来公平对比，为2+4x12。

![image](https://user-images.githubusercontent.com/15451867/114336107-35c72300-9b89-11eb-8a85-27ecfdd3bdd4.png)

若为四阶，则是2+8x6。 如此一来，高阶办法的弊端就得到了极大的缓解。而即便在这种情况下，这种堆叠方式同样可以带来显著的提升。

![image](https://user-images.githubusercontent.com/15451867/114335973-ebde3d00-9b88-11eb-850b-4082680acd54.png)
这种提升在浅层的时候差距很小，但随着深度的增加逐渐拉大，我们观测到的这个现象，也是符合上面不同阶数值方法的误差情况的。

在具有一定深度的情况下，不论是具体性能还是收敛的情况，都随着方法的阶数上升而获得了相应的提升。大家总说训网络是玄学，炼丹，这次能有这么清晰，性能按阶数逐步增强，差距随深度加深变大，这么符合理论的结果出现，说实话我个人也是吃了一惊。
![image](https://user-images.githubusercontent.com/15451867/114336255-7d4daf00-9b89-11eb-8115-65d8310da847.png)

![image](https://user-images.githubusercontent.com/15451867/114336500-1f6d9700-9b8a-11eb-93b3-34f929856cef.png)

## 有关实验的一些事情

#### 嵌套
堆叠的高阶block可以看做layer继续嵌套建立全局关系，如此一来网络设计就不是sub-net x N的格式，而是整体设计。单纯的堆叠对比，高阶办法的提升很明显，这让我们觉得也许block堆叠越少越好。但直接嵌套的方法并没有取得特别好的效果，早期的收敛加快了，但最后的性能基本上在误差浮动范围之内，没有观察到区别。

#### 自适应系数
二阶，三阶和四阶方法是固定系数的，二分之一，三分之二，二加减根号二之类的。在更高阶方法中，block内的中间状态在shortcut出去的时候会有抑制系数。这个系数会根据每次迭代返回的误差值动态调整。我们目前的实验中暂时还没有找到一个特别好用的变化方法。个人感觉这种形式和SEblock以及多头注意力都有点联系。

#### 数据集
数据集确实太小了，我太穷了嗷。因为要对比不同深度的几个不同阶方法，cifar-10基本上是极限了。常见的数值方法还有好几种，现在还在跑着呢。ImageNet只能指望别人去验证了。目前在Cifar-10上确实非常理想。

#### 下游任务
下游任务其实也试过好几个，但由于负担不起所以没能写进文章里。对U-Net是有提升的，对transformer基本上没有。我个人觉得一可能是因为多头注意力和输入抑制非常类似，再加上我跑的transformer堆叠block也不多，所以没有观察到明显提升。

#### 重参数化
二阶方法对RepVGG是有提升的，但更高阶就没有了。说明NN中的高阶堆叠还是依赖非线性的。这和一个block含有几层layer比较好的结果类似，2-3层比较理想，多了少了都会降。


[1] https://arxiv.org/pdf/1806.07366.pdf
[2] https://arxiv.org/pdf/1710.10121.pdf
