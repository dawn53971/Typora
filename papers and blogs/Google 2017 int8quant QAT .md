## contributions

1. 提供了8-bit 整数的量化网络，其中包含了权值和激活 为int8， bias（bias vector） 为32-bit
2. 在支持整数运算的硬件上（高通芯片）上实现，并且在附录里提供了在ARM上实现的具体描述。
3. 提供了QAT的框架（首次提出）

## Quantized Inference

$$
r = S(q-Z)
$$

通用量化表达式，对于$N\times N$ 的矩阵，对应于量化参数（$S_\alpha, Z_\alpha$) ,有
$$
r_\alpha ^{(i,j)} = S_\alpha (q_\alpha ^{(i,j)} - Z_\alpha). \\
S_3(q_3 ^{(i, k)} - Z_3) = \sum_{j=1} ^{N} S_1 (q_1 ^{(i, j)} - Z_1) \sum_{j=1} ^{N} S_2 (q_2 ^{(i, j)} - Z_2) ,\\ 
q_3 ^{(i, k)} = Z_3 + M \sum_{j=1} ^{N} (q_1 ^{(i, j)} - Z_1) \sum_{j=1} ^{N}(q_2 ^{(i, j)} - Z_2) ,\\
where\ \ \ M:= \frac {S_1S_2} {S_3}.
$$
对于这里唯一的非整数 M， 可以提前算好（offline）	$M = 2^{-n}M_0$ . 因此整个过程都是定点计算、

假如此刻M=0.007247473418460，P=7091，M与P相乘，相当于浮点数与整数相乘，仍会很吃资源。通过将M与$2^{-n}M_0$ 替换，通过脚本代码，算出最小误差时n为11， $M_0$ 为15.

此时， M*P就可以替换为$2^{-11} \times 15 \times 7091$ 

<img src="https://pic2.zhimg.com/80/v2-aed99e1940ff14ffe7b0d0d223a26711_720w.jpg" alt="img" style="zoom:80%;" />

<img src="https://pic3.zhimg.com/80/v2-c71f214c7bddac66d703fbf86f079086_720w.jpg" alt="img" style="zoom: 80%;" />

####  去除零点的处理复杂度（$O(N^3)$)

对于之前的式子
$$
q_3 ^{(i, k)} = Z_3 + M \sum_{j=1} ^{N} (q_1 ^{(i, j)}{\color {red} -Z_1)} \sum_{j=1} ^{N}(q_2 ^{(i, j)} {\color{red} - Z_2}) ,\\
$$
我们需要进行额外的$O(N^3)$ 的加法操作（零点对准）,为了简化，把这个式子变换一下
$$
q_3 ^{(i, k)} = Z_3 + M ( NZ_1Z_2 - Z_1 a_2 ^{(k)} - Z_2 \overline a_1 ^{(i)} + \sum_{j=1} ^{N}q_1 ^{(i, j)} q_2 ^{(i, j)} ),
$$
这里  
$$
a_2 ^{(k)} := \sum_{j=-1}^{N} q_2 ^{(j, k)}, \ \  \overline a_1 ^{(i)}:= \sum _{j=1}^{N} q_1 ^{(i, j)}.
$$
只有$O(N_2)$的额外次加法计算复杂度，相较于$\sum_{j=1} ^{N}q_1 ^{(i, j)} q_2 ^{(i, j)}$ 可以忽略



## Implementation of a typical  fused layer

在ARM，x86 CPU上，利用了gemmlowp library [Github location](https://github.com/google/gemmlowp) 

将q1矩阵作为权重，并将q2矩阵作为激活。权重和激活都属于uint8类型（可以等效地选择int8，并带有适当修改的零点）。uint8数值乘加需要一个32位累加器，为累加器选择一个带符号类型。$int32 = uint8 + uint8$ 

为了实现量化的偏置加法，需要对bias vector进行量化为int32，偏置矢量的量化缩放标度$S_{bias}$ 与累加器相同，
$$
S_{bias} = S_1S_2, Z_{bias} = 0
$$

## Training with simulated quantization

1. #### learning quantization ranges

2. #### Batch normalization folding

   BN在训练和推理时在计算图中是不同的， 推理时为了加速需要把BN fold进计算层。为了在训练时更加准确地模拟这一折叠BN的效果，来实现更好的QAT，需要对folding BN也进行simulate（结合量化）。We do with the following :
   $$
   \omega_{fold} := \frac{\gamma \omega}{\sqrt{EMA(\sigma_B ^2)+ \epsilon}}
   $$
   这里 $\gamma $ 是BN的scale 参数，$EMA(\sigma_B ^2)$ 是moving average estimate of 激活量化range统计时的卷积输出方差，$\epsilon$ 是常数来保持式子稳定（不除零）。（图示见论文附录）

   <img src="C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20211111230503951.png" alt="image-20211111230503951" style="zoom:67%;" /><img src="C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20211111230414723.png" alt="image-20211111230414723" style="zoom: 67%;" /> 

   <img src="C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20211111231039545.png" alt="image-20211111231039545" style="zoom:80%;" />

   

