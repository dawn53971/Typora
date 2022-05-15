==1==

Hi, my name is glm and i‘m a master student at SJTU for computer science.    I’m very happy to present to you of our work on mixed-precision quantization on U-Net for medical image segmentation.

==2==

Here is the outline of my presentation today.  

==3==

First is a quick introduction of U-Net and quantization .  U-Net is widely used for medical image segmentation. The increasing model complexity for improvement and diverse deployment requirements raise the demands for u-Net compression.  

Quantization is an effective method for neural network compression. It can reduce the model size and memory requirements by quzntizing the weights and activations. But current methods for low bitwidth quantization suffer from severe perfromance degradation. Our target is to quantize U-Net model for the task of medical image segmentation that separates tissues and organs in a high-precision manner.

==4==

As we konw, Quantization on weights and acitvations usually cause the model degradation. Regarding general purpose methods for network quantization, there are two popular ways for improving quantized models, i.e., mixed-precision bitwidth for better trade-off (over compression ratio and performance ) 2nd,  finetuning the quantized parameters to recover model accuracy.

Considering these, the problem is how to assign bitwidth and how to finetune quantized model with quantization noise.

==5==

I’ll  begin our work with the analysis of information flow in U-Net from a decoupled spatial frequency. Here is a gereral U-Net architecture,  There are two objectives in medical image segmentation, the consistency of semantic information, and preserving as much detail information as possible for accurate edge segmentation. The two objectives correspond to the low and high-spatial frequency information separately

 In the encoder part, large scale feature maps are contracted to small  through down-sampling to capture the global summation of low-spatial frequency. Correspondingly, up-sampling is adopted for expanding feature maps in decoder. However, down-sampled feature leads to high-spatial frequency loss, which cannot be recovered by up-sampling. For this reason, skip connection plays a key role in recovering the lost information, which bridges the contracting path and expansive path for forward pass of shallow features to decoder.



However, quantization severely disturbs this mechanism, resulting in a significant accuracy degradation. In particular, low-precision quantization of activation blurs the high-spatial frequency information of feature maps propagated on skip connection, causing the distortion of prediction.

==6==

 To verify our analysis, we implement activation quantization on the first skip connection with the rest modules untouched. We can see, Compared with the full precision model, quantization leads to blurred edges colored red and blue, which represent the false negative and false positive of pixel prediction. The eroded edges prove that high-spatial frequency information propagating on skip connection is important for accurate edge segmentation.

==7==

Based on the above analysis, we propose split convolution, which is possible to address the difference of spatial frequency.  It can be formulated by the equation of this. We split the convulution into two part for computing different feature maps of high/low spatial frequency.

After that, we  adopt the second formula to assign bitwidth, in which omiga represents the sensitivity of layer.  The higher omiga, we set the higher bitwidth.  

==8==

Our second proposed method is skip-supervised QAT, Focusing on the quantization aware training,  our skip supervision differs from previous works in several respects,  the quantization noise and blurred skip features.

**Quantization Noise**:   Quantization noise blocks the further improvement in finetuning. The coarse approximation of STE causes gradient noise in back propagation. This prevents the shallow layer from being trained effectively,  and leads to underfitting problem. Moreover, the coarse approximation of gradients make it more challenging for training shallow layers. 

2nd is **Blurred Skip Features:**    As we described above, down-sampling results in high-spatial frequency loss, which should be restored by skip connection. However, quantization of activation blurs the high-spatial frequency on skip connections, resulting in the distortion of edge segmentation.

==9==

Considering the two points, we design SS-QAT to finetune quantized network to a better performance. The auxiliary supervision branch is composed of a convolution followed by sigmoid function for predicting the ground truth. Worth mentioning that the skip supervision only works in quantization aware training, and can be removed in inference without affecting the final prediction. 

our supervision module is different from some traditional work like this(click).

This supervision boosts the efficiency of quantization aware training by providing additional gradients for shallow layers in back propagation. On the other hand, it improves the representation capability of encoders by producing more semantically meaningful feature for decoder.

(and with the split convolution, this composes our quantization scheme)

==10==

first, ... then,...after that..., finally 

==11==

The first table summarizes the comparison results. And second table shows the ablation study results. From these we can see that our mixed precision quantization and SSQAT help relieve the model degradation. 

==12==

Here exhibits the U-net model complexity and Hessian of each layer. Convolution is split into skip-conv and up-conv which are colored red in the figure. , which coresponding to the skip-connection and up-sampling.

From the figure we can see that  The up-conv is twice as computationally intensive as the skip-conv. Besides, as visually exhibited in second Fig. , Hessian trace of activations on skip-conv is much higher than that on up-conv (1.44∼3.11x), It indicates that up-conv is less sensitive to quantization, meaning a lower bitwidth for it. With regards to the two points, convolution with higher computation can be quantized to a lower bitwidth, which proves that a fine-grained bitwidth allocation make a difference for a better trade-off.

==13==

This is the qualitative comparisons of fixed-point quantization and ours. The quantization bitwidth of weights and activations are 3bit and 4bit.   This fig visualizes the segmented results, our method generates more accurate edge compared with a uniform  quantization, which proves that our method preserves more high-spatial frequency.

==14== 

At last is a short summary : we propose a novel quantization scheme including split convolution and skip supervision for QAT, which achieves a fine-grained mix precision bitwidth and efficient quantization aware training.







