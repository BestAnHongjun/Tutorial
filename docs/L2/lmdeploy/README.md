# LMDeploy量化部署进阶实践

![](./figures/topic.jpg)

## 1.准备工作

### 1.1 创建开发机
本节课程所有实验将使用InternLM-2.5-1.8B模型作为示例。1.8B模型，以FP16/BF16格式加载模型，至少需要占用 $(1.8 \times 2)=3.6$ GB的显存。因此，我们使用一个具有8G显存的开发机足够运行该模型。

在InternStudio平台，创建一个10% A100 (cuda12.2-conda)的镜像。详细操作步骤请参考L1课程。

![](./figures/internstudio-image.jpg)

### 1.2 配置虚拟环境

首先创建一个新的conda环境，名为`lmdeploy`，解释器版本为`Python 3.10`。

```sh
conda create -n lmdeploy python=3.10
conda activate lmdeploy
```

安装`pytorch`模块。由于后续我们要安装`lmdeploy v0.6.1`，依赖[`torch<=2.3.1`](https://github.com/InternLM/lmdeploy/blob/v0.6.1/requirements/runtime.txt)，因此我们安装2.3.1版本的`pytorch`。

```sh
pip install torch==2.3.1 torchvision==0.18.1 torchaudio==2.3.1 --index-url https://download.pytorch.org/whl/cu121
```

## 2.KV Cache缓存推理实验


### 1.1 LLM前向推理过程回顾

首先回顾一下LLM的前向推理过程。为了突出本章重点内容，省略了一些关于位置编码、Normalize算子的计算细节。

对于现有输入token序列 $T_{1:n}=[t_1, t_2, ..., t_n]$ ，首先会经过嵌入层(Embedding Layer)得到嵌入向量，这同时也是LLM第1层block的输入：

$$X_{1:n}^{(1)}=\text{EMB}(T_{1:n})=[x_1^{(1)}, x_2^{(1)}, ..., x_n^{(1)}]$$

该嵌入向量会依次经过LLM若干层block。对于模型第 $l$ 层，输入嵌入向量 $X_{1:n}^{(l)}=[x_1^{(l)}, x_2^{(l)}, ..., x_n^{(l)}]$ ，会先经过三个线性变换矩阵  $W_q^{(l)}$ 、 $W_k^{(l)}$ 、 $W_v^{(l)}$ ，得到 $Q_{1:n}^{(l)}$ 、 $K_{1:n}^{(l)}$ 、 $V_{1:n}^{(l)}$ ，如下：

$$Q_{1:n}^{(l)}=W_q^{(l)} X_{1:n}^{(l)}=[q_1^{(l)}, q_2^{(l)}, ..., q_n^{(l)}]$$

$$K_{1:n}^{(l)}=W_k^{(l)} X_{1:n}^{(l)}=[k_1^{(l)}, k_2^{(l)}, ..., k_n^{(l)}]$$

$$V_{1:n}^{(l)}=W_v^{(l)} X_{1:n}^{(l)}=[v_1^{(l)}, v_2^{(l)}, ..., v_n^{(l)}]$$

经注意力汇聚过程，得到输出 $H_{1:n}^{(l)}$ ：

$$H_{1:n}^{(l)}=\frac{\text{softmax}(Q_{1:n}^{(l)\top}\cdot K_{1:n}^{(l)})}{\sqrt{d_k}}V_{1:n}^{(l)}=[h_1^{(l)}, h_2^{(l)}, ..., h_n^{(l)}]$$

经过前馈层，得到本层输出 $Y_{1:n}^l$ ，计算细节略：

$$Y_{1:n}^{(l)}=\text{FFN}(H_{1:n}^{(l)})=[y_1^{(l)}, y_2^{(l)}, ..., y_n^{(l)}]$$

$Y_{1:n}^{(l)}$ 会作为下一层block的输入 $X_{1:n}^{(l+1)}$ ，即：

$$X_{1:n}^{(l+1)} = Y_{1:n}^{(l)}$$

对于最后一层（第 $L$ 层）的输出，最后一个向量 $y_n^{(L)}$ 将会经过一个线性层，转换为一个词表长度vocab_size大小的概率向量。随后经过一定的采样策略，以贪心采样策略为例(每次都选概率最大)，生成新的token。

$$t_{n+1} = \text{argmax}(\text{Linear}(y_n^{(L)}))$$

### 1.2 KV Cache技术

新的 $t_{n+1}$ 会和 $T_{1:n}$ 组合，作为新一轮的输入，然后计算每一层新的 $Q_{1:n+1}^{(l)}$ 、 $K_{1:n+1}^{(l)}$ 、 $V_{1:n+1}^{(l)}$ ：

$$Q_{1:n+1}^{(l)}=W_q^{(l)} X_{1:n+1}^{(l)}=[q_1^{(l)}, q_2^{(l)}, ..., q_{n}^{(l)}, q_{n+1}^{(l)}]$$

$$K_{1:n+1}^{(l)}=W_k^{(l)} X_{1:n+1}^{(l)}=[k_1^{(l)}, k_2^{(l)}, ..., k_{n}^{(l)}, k_{n+1}^{(l)}]$$

$$V_{1:n+1}^{(l)}=W_v^{(l)} X_{1:n+1}^{(l)}=[v_1^{(l)}, v_2^{(l)}, ..., v_{n}^{(l)}, v_{n+1}^{(l)}]$$

我们发现，其中 $[q_1^{(l)}, q_2^{(l)}, ..., q_{n}^{(l)}]$ 、 $[k_1^{(l)}, k_2^{(l)}, ..., k_{n}^{(l)}]$ 和 $[v_1^{(l)}, v_2^{(l)}, ..., v_{n}^{(l)}]$ 在上一轮已经计算过一遍了，这里出现了**计算冗余**。为了避免冗余的计算，我们不妨把上一轮计算过的 $Q_{1:n}^{(l)}$ 、 $K_{1:n}^{(l)}$ 、 $V_{1:n}^{(l)}$ “**缓存**”起来，本轮计算变为如下形式：

$$Q_{1:n+1}^{(l)}=W_q^{(l)} X_{1:n+1}^{(l)}=[Q_{1:n}^{(l)}, W_q^{(l)}x_{n+1}^{(l)}]=[Q_{1:n}^{(l)}, q_{n+1}^{(l)}]$$

$$K_{1:n+1}^{(l)}=W_k^{(l)} X_{1:n+1}^{(l)}=[K_{1:n}^{(l)}, W_k^{(l)}x_{n+1}^{(l)}]=[K_{1:n}^{(l)}, k_{n+1}^{(l)}]$$

$$V_{1:n+1}^{(l)}=W_v^{(l)} X_{1:n+1}^{(l)}=[V_{1:n}^{(l)}, W_v^{(l)}x_{n+1}^{(l)}]=[V_{1:n}^{(l)}, v_{n+1}^{(l)}]$$

这样，我们就得到了“**QKV Cache**”缓存方法。

再来观察采样过程。对于第n+1轮迭代，虽然模型输出了 $[y_1^{(L)}, y_2^{(L)}, ..., y_{n+1}^{(L)}]$ ，但其实只有 $y_{n+1}^{(L)}$ 是我们需要的结果。往前推，其实我们在注意力汇聚时，只需要得到 $h_{n+1}^{(l)}$ 。

$$h_{n+1}^{(l)}=\frac{\text{softmax}(q_{n+1}^{(l)\top} \cdot K_{1:n+1}^{(l)})}{\sqrt{d_k}}V_{1:n+1}^{(l)}$$

也就是说，我们其实只需要 $q_{n+1}^{(l)}$ 、 $K_{1:n+1}^{(l)}$ 、 $V_{1:n+1}^{(l)}$ 。对 $Q_{1:n}^{(l)}$ 的缓存也是不必要的。

因此，在LLM推理过程中，我们通常会对 $K_{1:n}^{(l)}$ 、 $V_{1:n}^{(l)}$ 进行缓存，称为**KV Cache**技术，通过避免重复计算来提升模型推理速度。当然，这通常会带来额外的内存开销。


## 2.大模型量化技术

### 2.1 通用量化原理

#### 量化过程

**量化**通常是指将模型参数由较多比特的机器数映射为较少比特的机器数的过程，如将fp16量化为fp8/int8/int4等。以将浮点数量化为整数为例，一组浮点数 $X$ ，量化后的定点数 $X_q$ 可由如下方式计算：

$$X_q= \lfloor \frac{X}{\Delta} \rfloor$$

其中， $\Delta$ 称为量化单位，计算方法如下：

$$\Delta=\frac{\text{max}(|X|)}{2^{N-1}}$$

$N$ 为量化后机器数的比特数。

#### 反量化过程

**反量化**则是将较少比特的机器数映射为较多比特的机器数的过程。

$$X'=\Delta X_q$$

#### 量化技术的特点

* 通常来讲，量化是有精度损失的，反量化是无精度损失的。
* 量化技术可以减少参数占用的内存空间，**但不一定能起到加速效果**。
* 量化技术的加速收益通常来源于两个方面：
  * **硬件计算单元的加速支持**：有的硬件支持对低比特机器数运算进行加速，如NVIDIA GPU中的int8计算单元。如果没有硬件的支持，或者推理时数据类型不满足硬件的加速条件，则不能获得明显的速度收益(如AWQ量化技术，仅将参数进行int4存储，但计算激活值时还会反量化为fp16)；
  * **节省的内存带宽**：有的任务属于访存瓶颈任务(即运算单元利用率不高，大部分时间花费在显存<->运算单元寄存器的搬运上)，通过量化减少参数占用的内存，从而减轻IO带宽压力，提升运算单元利用率，获得速度收益。

### 2.2 AWQ量化技术

[[详解传送门]](https://zhuanlan.zhihu.com/p/697761176)

### 2.3 SmoothQuant量化技术

[[详解传送门]](https://zhuanlan.zhihu.com/p/703928680)

## 3.大模型外推技术

### 3.1 什么是外推

大语言模型的外推性问题指的是大模型在训练时和预测时的输入长度不一致，导致模型的泛化能力下降的问题。这一问题主要来自两个方面：

* 预测阶段用到了没训练过的位置编码
  * 模型不可避免地在一定程度上对位置编码“过拟合”
* 预测时，注意力机制所处理的token数量远超训练时的数量
  * 导致计算注意力时的“熵”差异较大

### 3.2 位置编码外推



### 3.2 注意力缩放外推

## 4.Function Calling功能

## 5.实验课

请参考[experiment.md](./experiments.md)。