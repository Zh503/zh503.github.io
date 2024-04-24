---
title: csgvd笔记
date: 2024-04-24 15:37:38
categories:
  - 笔记
tags:
  - BiLSTM
  - CFG
  - Devign
---

##  CSGVD

将漏洞检测视为函数级图二分类问题,

- 提出CSGVD模型
- 提出PE-BL模块来初始化嵌入,使用BiLSTM聚合语义信息
- 提出M-FBA(mean biaffine attention pooling均值双仿射注意力池化用于图嵌入)

![image-20240422105526958](https://s2.loli.net/2024/04/22/VaIhbLZ6yPRmckX.png)

### 第一步:特征提取

使用joern对代码CFG进行提取,避免CFG过大,通过行来合并提取的节点,压缩图形，并将原始的控制流图转换为语句级的控制流图,压缩后的图中的节点数量将少于或等于代码行数，这将大大减少计算时间和资源。

### 第二步:节点嵌入

输入为源代码的单个函数,拆分为一组语句$S = \{s_1, s_2, s_3, . . . , s_n\} $,使用预训练CodeBert的来初始化嵌入层.

使用CodeBert中的PBE标记器进行标记,如公式(1)

然后用CodeBert的词嵌入层权重初始化权重的词嵌入层来获得每个标记的向量表示$e_{i j}\ \in\ E_{i}\,=\,\{e_{i1},\,e_{i2},\,e_{i3},\,\cdot\,\cdot\,,\,e_{i n}\}$：
$$Ei = Embedding(tokensi)。$$
其中CodeBERT使用的字节对编码（BPE）（Sennrich等人，2016）标记器也可以缓解OOV问题。
$$tokensi = BPE − Tokenizer(si)。$$

$$Ei = Embedding(tokensi)。$$
最后使用BiLSTM来融合代码的本地语义信息
$$
{h}_{i},h_{i}\,=\,BiLSTM(E_{i})
$$

$$
x_i=Sum(\overrightarrow{h}_{i},\overleftarrow{h_{i}})
$$

其中$\overrightarrow{h}_{i},\overleftarrow{h_{i}}$是BiLSTM 的最终输出

### 第三步:残差图神经网络

有了CFG和代码的语义信息,就需要对信息进行融合.

构建一个残差图神经网络来对语义信息和CFG控制流图进行融合
$$H^{(l+1)} = GCN (H^{(l)}, A) + H^{(l)}$$
$$Specially, in CSGVD,$$
$$H^{(0)} = X$$
$$H^{(1)} = GCN(H^{(0)}, A) $$
$$\text {where in}\quad X = {x_1, x_2, x_3, . . . , x_n}\quad  \text{is  the node embedding matrix}.$$
![image-20240422110131759](https://s2.loli.net/2024/04/24/gu48XHO9QKF6UMh.png)

### 第四步:图嵌入

作者提出了平均双仿射注意（M-BFA）池化方法,通过注意力机制来学习语句的权重.

![image-20240422105752665](https://s2.loli.net/2024/04/22/CtRrHknFvw2KiNP.png)

作者假设存在一个超节点vs，它在图的信息聚合中起着主导作用。计算每个节点与vs之间的注意力分数。最后，对每个节点进行加权聚合，作为图的向量表示。
$$h_{m e a n}=\frac{\sum_{i=1}^{n}h_{i}}{n}$$

$$h_{fi}=h_{i}^{T}\cdot W$$

$$e_{i}=h_{fi}^{T}\cdot h_{m e a n}+h_{i}^{T}\cdot u$$

$$a_{i}=\frac{\exp{(e_{i})}}{\sum_{j=0}^{n}\exp{(e_{j})}}$$

$$h_{g}=\sum_{i=0}^{n}a_{i}\cdot h_{fi}$$

其中，W 是一个可学习的权重矩阵，u 是一个可学习的权重向量，hi 是节点 vi 的最终隐藏状态，而 hg 表示图的向量表示。特别地，我们选择所有节点的隐藏表示的平均 hmean 作为超级节点 vs 的隐藏表示。

### 第五步:分类器学习

这一步就是个简单的分类,。构建一个两层MLP，如方程（12）–（13）所示，来预测一个给定函数是否存在漏洞。
$$o = softmax(σ (U \cdot h_g ) \cdot V )$$
$$y = argmax(o)$$
其中U和V是两个可学习的权重矩阵，σ(·)是一个激活函数，而y∈{0, 1}表示最终的预测结果。

### 数据集

使用Devign数据集

我们构建了一个2层图神经网络模型，将batch size设置为128，dropout设置为0.5，并使用Adam优化器和线性学习率调度器来训练模型，learning rate为5e−4，最多可训练100epochs。所有隐藏层 LSTM, GNN, and MLP, 隐藏大小设置为 768,等于预训练的嵌入的维度。

### 实验结果

#### 检测效果对比

![image-20240424160947295](https://s2.loli.net/2024/04/24/2Tov3O4jdxDZFXt.png)

#### 不同的节点嵌入和初始化方法会如何影响模型性能？

![image-20240424161026419](https://s2.loli.net/2024/04/24/VlCGYZaD3jWkLPh.png)

#### 不同的令牌融合方法如何影响模型性能？

![image-20240424161231502](https://s2.loli.net/2024/04/24/aTE2tGOfX69vRhF.png)

#### 不同的图神经网络模型如何影响模型性能？

![image-20240424161402185](https://s2.loli.net/2024/04/24/CmlRkwuL5xtMgEY.png)

#### M-BFA对图嵌入的影响

![image-20240424161427946](https://s2.loli.net/2024/04/24/Lays1Yio5l4NPT6.png)