# AdaBits

本质上不算是混合精度量化，而是固定bit-width量化的动态延拓。作者受到两篇paper的启发，一个是Slimmable neural networks，在多种width配置下进行一次训练，然后根据应用场景任意切换剪枝策略而无需重新训练，另一篇就是Once For All，这篇论文对不同配置的宽度，深度和卷积核的大小都进行了渐进训练，使得一次训练完之后，可以根据应用场景从大模型中任意剥离一个小模型出来部署，而无需再训练。本文的核心思想类似，对一个模型的不同量化策略进行同时训练，然后根据应用场景，可以任意切换到某个确定bit-width下进行部署而无需再次finetune



## Related Works

用了PACT和SAT两种方法作为基础

## SAT（Towards Efficient Training for NN Quantization)：

==Scale-adjusted training==(提出的主要观点：量化是因为训练不足导致的性能下降，而不是网络的容量降低导致，量化提供一种正则手段。)

角度：梯度回传 -> 提供了两个准则: 

同时分析了PACT计算梯度时的量化噪声

### ==Efficient Training Tule==

1. 为了防止输出层交叉熵计算时进入==饱和==区域，输出层（全连接层）的权值的scale（measured by the variance) 应该尽可能小，
   $$
   \mathbb {VAR}[\Xi_{ij} ^{(L)}] \ll \frac{(k_{L-1}^{pool})^2}{n_L}
   $$
   $n_L$指全连接输入的特征长度，$k_{L-1}^{pool}$指前一层pooling layer的kernel大小（empirically, 小于0.1就足够了 ）。

2. > To avoid gradient exploding/vanishing problems, the magnitudes of gradients should be on the same order when propagating from one layer to another.

   为了使整个网络的权重梯度scale保持一致，要么在线性层之后使用BN层，如卷积层和全连接层，要么有效权重的方差应该是(`on the same order? `)线性层神经元数量的倒数。

通过以上两条准则，来测试量化模型的训练是否efficient。权值量化：DoReFa   激活量化： PACT



### Weight Quantization with SAT

DoReFa shceme involves two steps: clamping + quantization

> clamping transforms the weights to values between 0 and 1
>
> quantization rounds the weights to integers.

##### Impact of clamping

Clamping transformation:
$$
\widetilde {W_{ij}} = \frac{1}{2} (\frac{tanh(W_{ij})}{max_{r,s}|tanh(W_{rs})|}+1) \\
\widehat {W_{ij}} = 2 \widetilde W_{ij} -1
$$
压缩了大的权值，增大了小权值的差异，因此使得权值的分布更加均匀了，（在【0，1】内），有利于减少量化误差。**但是导致的后果是违反了前面两个准则**

> As a common practice [13], the original weights W are sampled from a Gaussian distribution of zero mean and variance proportional to the reciprocal of the number of neurons. We find that for large neuron numbers, the variance of weights can be enlarged to tens of their original values, potentially increasing the opportunity of violating ETR I. Since the number of output neurons of the last linear layer is determined by the number of classes of data, we expect large dataset such as ImageNet [7] to be more vulnerable to this problem than small dataset such as CIFAR10

*原本的权值满足高斯分布采样，均值0，方差与输出神经元个数的倒数正比* 。现在Clamping之后，随着神经元数目增多，方差显著大于原始权值的方差

因此引入了Scale-adjusted training(SAT) 来恢复权值的方差。constant rescaling $\mathbb {VAR}[\widehat {W_{rs}}]$ 被视为常数，不回传梯度。
$$
W_{ij} ^{*} = \frac{1}{\sqrt{\hat n \mathbb {VAR}[\widehat W_{rs}]}} \widehat W_{ij}
$$

##### Impact of weight quantization

$$
q_k(x) = \frac{1}{a} \lfloor ax \rceil  \\
Q_{ij} = 2 q_k(\widetilde W_{ij}) -1
$$

当权值clamped [0, 1]时，进一步的量化函数 $\lfloor \bullet \rceil$ 表示四舍五入取整操作。

> We find that for precision higher than 3 bits, quantized weights have nearly the same variance as the full-precision weights, indicating quantization itself introduces little impact on training. However, the discrepancy increases significantly for low percision, thus we should use the variance of quantized weights $Q_{ij}$ for standardization, rather than that of the clamped weight $\widehat W_{ij}$.

低比特量化使得量化后权值的方差发生了变化，需要重新标准化。

**==Key equation==** 
$$
Q_{ij} ^{*} = \frac{1}{\sqrt{n_{out} \mathbb {VAR}[Q_{rs}]}} Q_{ij}
$$

### Activation Quantization with CG-PACT

###### > PACT 

>  训练截断relu的$\alpha$ + quantization, 反向传播时利用STE来计算y对$\alpha$的导数。

$$
y_q = round(y \cdot \frac{2^k -1}{\alpha}) \cdot \frac{\alpha}{2^k -1}\\
\frac{\partial y_q}{\partial \alpha} = \frac{\partial y_q}{\partial y} \frac{\partial y}{\partial \alpha}= \begin{cases} 0, & x \in (-\infty, \alpha) \\ 1, & x \in [\alpha, +\infty)  \end{cases}
$$

Different from original PACT,  Calibrated gradient
$$
\frac{\partial y_q}{\partial \alpha} = \frac{\partial y_q}{\partial y} \frac{\partial y}{\partial \alpha}= \begin{cases} q_k(\frac{\tilde x}{\alpha}) -\frac{\tilde x}{\alpha}, & x \in (-\infty, \alpha) \\ 1, & x \in [\alpha, +\infty)  \end{cases}
$$
与原始的忽略了量化噪声的方式不同 i.e., ==$q_k(\frac{\tilde x}{\alpha}) \approx \frac{\tilde x}{\alpha}$==

对于低比特量化中这个噪声不可以被忽略。



-----------

********

**Revisiting Scale-Adjusted Training (SAT)**

> The ket idea is that quantized models usually enforce large variance in their weights, which brings about over-fitting issue during training.

## Quantization with Adaptive Bit-widths(method)

### Modified DoReFa Scheme

对原始DoReFa的$q_k(x) = \frac{1}{a} \lfloor ax \rceil$ 的rounding 改为了取floor，==$q_k(x) = \frac{1}{a} min(\lfloor \widehat ax \rfloor , \widehat a - 1)$== 

以保证不同bit-width量化后的数值存在严格的倍数关系

![image-20211115135414971](C:\Users\admin\Documents\Typora\typora-pics\image-20211115135414971.png)

作者探讨了三种自适应bit-width的方法

### Direct Adaptation

 第一种方法是在某个固定bit下进行量化训练，然后直接切换到其他bit-width上，发现效果不行，根据一些文献发现是BN统计值在不同bit-width下不匹配的问题，于是引入了==BN参数校准==来解决这个问题，发现效果不错，但离单个精度下训练的方式还是有些差异，作者认为原因是训练阶段和量化测试阶段的设置不一致导致的。*训练和测试的gap*

<img src="C:\Users\admin\Documents\Typora\typora-pics\image-20211115165510436.png" alt="image-20211115165510436" style="zoom:80%;" />

### Progressive Quantization

Refer to a quantization model trained on multiple bit-widths sequentially

第二种是渐进式的训练量化，有两种实现方式，一种是从高位宽到低位宽，一种相反。

实现方式： 先训练一个初始位宽的量化模型，然后逐步提高/降低位宽并finetune。（包含BN参数校准）

通过finetune来消除训练和测试的gap，实验证明这相对第一种直接改变位宽的方式效果有所提升。低bit到高bit的训练策略最终在低bit的结果会很差，而高bit到低bit的训练结果与单个精度下训练很相近，但仍然还差那么一点点。

<img src="C:\Users\admin\Documents\Typora\typora-pics\image-20211115164819235.png" alt="image-20211115164819235" style="zoom:80%;" />

### Joint Quantization

1. simultaneously train models under different bit-widths with shared weights
2. switchable batch normalization 
3. switchable clipping level

> Instead of training models with different channel numbers, we simultaneously train models under different bit-widths with shared weights. Also, as mentioned above, quantization with different bit-widths leads to different variances of quantized weights and activations. Based on this, we adopt the switchable batch normalization technique(used in slimmable )

第三种就是联合量化法，上个按顺序的渐进训练不行，会引入不必要的干扰，那么就让所有的bit-width同时参与训练，类似于==slimmable neural networks==，提出了==Vanilla AdaBits==的方法，同时采用了`switchable batch normalization`, 

这里没有列出2-4bit的实验结果，而是4-8bit，实验结果很接近单个精度下训练的结果，甚至在5-6bit实现了超越，但在4bit的地方还是差了0.5个点(太苛刻了吧)

<img src="C:\Users\admin\Documents\Typora\typora-pics\image-20211115170750706.png" alt="image-20211115170750706" style="zoom:80%;" />

作者通过分析得出是PACT那个可学习的α阈值的问题，不同bit-width共享的同一个α阈值，而实际单个精度下训练进行统计发现，不同精度的阈值是有较大差异的，而且精度越高，这个阈值会越大。在受到switchable Batch Normalization的启发下，作者提出了第三种方法，即对每个bit-width都学习一个对应的α，每次量化的时候都使用自己的那个α，互不干扰，而且增加的这一点点参数量与整个相比是微不足道的，但确实可以解决共享α带来的问题



## Future work

1. 利用nas的策略，迁移到位宽选择上，实现混合精度位宽分配。



