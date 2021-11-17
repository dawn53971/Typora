# QUANTIZATION NOTES

## 知乎 量化基础 

## 量化训练和推理

[工程实现 ｜加速神经网络推理之 8bit Quantization （模型量化压缩） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/38328685) :bookmark_tabs:

最初的量化方法：

$V_q = Q * (V_x - min(V_x))$

$V_x' = V_q/Q + min(V_x)$

其中$Q = S/R, R = max(V_x)-min(V_x), S = 1 << bits-1$。  $V_x$ 表示原始浮点输入，$V_q$ 表示量化后的定点数值， $V_x'$ 是根据量化参数还原出的浮点数， bits 表示量化位宽，上述公式可以进一步简化为：
$$
V_q = round(Q*V_x)-RQM \\
V_x' = (V_q + RQM) / Q \\
RQM = round(Q*min(V_x))
$$
python 代码如下：

```python
# 根据参数和原始浮点数量化为定点数
def Quant(Vx, Q, RQM):
    return round(Q * Vx) - RQM

# 根据量化参数还原回浮点
def QuantRevert(VxQuant, Q, RQM):
    return (VxQuant + RQM) / Q
```

针对数组量化和还原：

```python
def ListQuant(data_list, quant_bits):
    # 数组范围估计
    data_min = min(data_list)
    data_max = max(data_list)
    # 量化参数估计
    Q = ((1 << quant_bits) - 1) * 1.0 / (data_max - data_min)
    RQM = (int)(round(Q*data_min))
    # 产生量化后的数组
    quant_data_list = []
    for x in data_list:
        quant_data = Quant(x, Q, RQM)
        quant_data_list.append(quant_data)
    quant_data_list = np.array(quant_data_list)
    return (Q, RQM, quant_data_list)

def ListQuantRevert(quant_data_list, Q, RQM):
    quant_revert_data_list = []
    for quant_data in quant_data_list:
        # 量化数据还原为原始浮点数据
        revert_quant_data = QuantRevert(quant_data, Q, RQM)
        quant_revert_data_list.append(revert_quant_data)
    quant_revert_data_list = np.array(quant_revert_data_list)
    return quant_revert_data_list
```



前几个小节主要借鉴的是2016年的文章[On the efficient representation and execution of deep acoustic models](https://link.zhihu.com/?target=https%3A//arxiv.org/abs/1607.04683)，然而，在2017年谷歌更近一步扩展到了全定点的计算， 参考 [Quantization and Training of Neural Networks for Efficient Integer-Arithmetic-Only Inference](https://link.zhihu.com/?target=https%3A//arxiv.org/abs/1712.05877)，原理和之前分享的内容也基本上一致，将32位的 RQM 换成了 8 位的 zero_point，具体的量化反量化操作如下代码：

``````pyth
def ChooseQuantizationParamsV2(xmin, xmax, num_bits=8):
    qmin = -(int)(1 << (num_bits-1))
    qmax = (int)((1 << (num_bits-1))-1)

    scale = (xmax - xmin) / (qmax - qmin)
    initial_zero_point = qmin - xmin / scale
    nudged_zero_point = initial_zero_point
    if initial_zero_point < qmin:
        nudged_zero_point = qmin
    elif initial_zero_point > qmax:
        nudged_zero_point = qmax
    else:
        nudged_zero_point = (int)(initial_zero_point)

    return scale, nudged_zero_point

def QuantizeV2(x, scale, nudged_zero_point, num_bits=8):
    qmin = -(int)(1 << (num_bits-1))
    qmax = (int)((1 << (num_bits-1))-1)
    transformed_val = x/scale+nudged_zero_point
    clamped_val = np.clip(transformed_val, qmin, qmax)
    quantized = np.round(clamped_val)
    return quantized.astype(np.int8)

def DequantizeV2(x, scale, nudged_zero_point):
    return (x.astype(np.float32) - (float)(nudged_zero_point)) * scale
``````



目前，既要保证识别效果，同时还要使用 8 bit 量化模型，一种比较完备的做法就是将推理阶段的量化操作迁移到训练阶段，如 Tensorflow 说明文档一章介绍 [Fixed Point Quantization](https://link.zhihu.com/?target=https%3A//www.tensorflow.org/performance/quantization)。采用 fake 的量化后的浮点来作为 input 和 weight 的替换，同时浮点范围采用了平滑最大最小值的方法，具体可以查看 TensorFlow 的官方代码 [MovingAvgQuantize](https://link.zhihu.com/?target=https%3A//github.com/tensorflow/tensorflow/blob/05ab6a2afa2959410d48aab2336cea9dc1e2c13e/tensorflow/contrib/quantize/python/quant_ops.py%23L186)，训练中平滑得到了全局的输入和权重的最大最小值（类似 batchnorm 的统计方式），代码段如下：

``````pyth
    if symmetric:
      if narrow_range:
        min_max_ratio = -1
      else:
        # In two's complement notation, the negative range is slightly larger
        # than the positive range.
        min_max_ratio = -((1 << num_bits) - 2) / (1 << num_bits)

      # TFLite requires that 0.0 is always in the [min; max] range. Because
      # batch_min <= batch_max, it follows that range_min <= 0 <= range_max.
      range_min = math_ops.minimum(batch_min, batch_max / min_max_ratio)
      range_max = math_ops.maximum(batch_max, batch_min * min_max_ratio)
    else:
      # TFLite requires that 0.0 is always in the [min; max] range.
      range_min = math_ops.minimum(batch_min, 0.0)
      range_max = math_ops.maximum(batch_max, 0.0)

    assign_min = moving_averages.assign_moving_average(
        min_var, range_min, ema_decay, name='AssignMinEma')
    assign_max = moving_averages.assign_moving_average(
        max_var, range_max, ema_decay, name='AssignMaxEma')

    return _FakeQuantWithMinMaxVars(
        inputs,
        assign_min,
        assign_max,
        per_channel=per_channel,
        num_bits=num_bits,
        narrow_range=narrow_range)
``````



:carousel_horse: : emoji

关于 int8t 的推理过程，还有一个官方博客把推理过程也写出来了 [TensorFlow Lite 8-bit quantization specification](https://link.zhihu.com/?target=https%3A//www.tensorflow.org/lite/performance/quantization_spec)，细节之处让人不得不佩服谷歌的匠心，除了因式分解后的提前计算，还有将权重的最大最小值强制设计成对称，这样 zero_point 等于零，减少了两次计算，如下图所示。

#### 关于定点/浮点的点积优化方法，可以看看下面几个开源项目：

- Google 开源的代码 [gemmlowp: a small self-contained low-precision GEMM library](https://link.zhihu.com/?target=https%3A//github.com/google/gemmlowp) ，结合 [Tensorflow](https://link.zhihu.com/?target=https%3A//www.tensorflow.org/) 的量化模型训练应该能发挥很好的『药效』。Google 很慷慨地公布了几种 [GEMM](https://link.zhihu.com/?target=https%3A//en.wikipedia.org/wiki/GEMM) 优化方法，比如基于 NEON / SSE 的 SIMD 指令、并行化运算、Cache 预热等优化方法。该 C++ 代码移植到目前市面上绝大部分的手持设备上应该不成问题。
- ARM 公司开源项目，针对自家嵌入式设计的词唤醒引擎架构 [ML-KWS-for-MCU](https://link.zhihu.com/?target=https%3A//github.com/ARM-software/ML-KWS-for-MCU) ，里面针对 **DS-CNN** / **DNN** 模型做了大量的定点计算优化工作，另外该项目还开放了基于 Tensorflow 的 8 位量化模型训练工具，移植到嵌入式设备上的底层优化算法库是来自自家的 [CMSIS_5](https://link.zhihu.com/?target=https%3A//github.com/ARM-software/CMSIS_5) 。
- 如果 GEMM 优化还没入门，还可以从 [OpenBlas](https://link.zhihu.com/?target=https%3A//www.openblas.net/) 贡献者在 Github 发布的指导文档 [How To Optimize Gemm](https://link.zhihu.com/?target=https%3A//github.com/flame/how-to-optimize-gemm) 开始入手。

更多针对移动端的开源代码：

| Framework  | Organizer | Link                                                         |
| ---------- | --------- | ------------------------------------------------------------ |
| MACE       | Xiaomi    | [https://github.com/XiaoMi/mace](https://link.zhihu.com/?target=https%3A//github.com/XiaoMi/mace) |
| FeatherCNN | Tencent   | [https://github.com/Tencent/FeatherCNN](https://link.zhihu.com/?target=https%3A//github.com/Tencent/FeatherCNN) |
| NCNN       | Tencent   | [https://github.com/Tencent/ncnn](https://link.zhihu.com/?target=https%3A//github.com/Tencent/ncnn) |
| TFLite     | Google    | [https://github.com/tensorflow](https://link.zhihu.com/?target=https%3A//github.com/tensorflow) |
| Caffe2     | Facebook  | [https://github.com/caffe2/caffe2](https://link.zhihu.com/?target=https%3A//github.com/caffe2/caffe2) |
| TNN        | Tencent   | [https://github.com/Tencent/TNN](https://link.zhihu.com/?target=https%3A//github.com/Tencent/TNN) |

