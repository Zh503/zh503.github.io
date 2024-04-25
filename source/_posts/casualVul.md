---
title: causalVul
date: 2024-04-22 09:53:28
categories:
  - 学习笔记
  - 论文译文
tags:
  - Big-Vul
  - Devign
author: 
  name: hao
  link: https://github.com/zh503
---

# Towards Causal Deep Learning for Vulnerability Detection

论文链接:https://arxiv.org/pdf/2310.07958v5.pdf

论文开源地址: https://figshare.com/s/0ffda320dcb96c249ef2.

使用数据集:Big-Vul,Devign

## 摘要

近年来，深度学习漏洞检测显示出了令人鼓舞的结果。然而，一个至关重要的挑战仍然妨碍其在实践中的大规模应用，即模型在受干扰时不具有鲁棒性，并且无法很好地处理超出分布（OOD）的数据，例如将训练好的模型应用于真实世界中看不见的项目。我们假设这是因为模型学习到了非鲁棒特征，例如变量名，这些特征与标签之间存在虚假相关性。当受干扰和OOD数据集不再具有相同的虚假特征时，模型的预测就会失败。为了解决这一挑战，在本文中，我们将因果关系引入到深度学习漏洞检测中。我们的方法CausalVul包含两个阶段。首先，我们设计了新颖的扰动来发现模型可能用来进行预测的虚假特征。其次，我们在现有的深度学习模型上应用因果学习算法，特别是do-计算，系统性地消除了对虚假特征的使用，从而促进了基于因果的预测。我们的结果显示，CausalVul显著提高了我们测试的所有最先进模型和数据集的模型准确性、鲁棒性和OOD性能。据我们所知，这是第一项将基于do-计算的因果学习引入软件工程模型中并显示其确实有助于提高模型准确性、鲁棒性和泛化性能的工作。我们的复现软件包位于https://figshare.com/s/0ffda320dcb96c249ef2。

## 引言

源代码漏洞指的是代码中可能存在的潜在缺陷，可能被外部攻击者利用来威胁系统的安全。在过去，漏洞已经造成了重大的数据和财务损失。尽管过去已经开发了许多自动漏洞检测工具，但漏洞仍然很普遍。仅在2023年，国家漏洞数据库就收到并分析了惊人数量的16,000个漏洞。美国政府的网络安全基础设施安全局（CISA）报告称，自2022年以来，已有超过850个已知漏洞在来自150多家公司的产品中被利用，其中包括谷歌、微软、Adobe和思科等主要科技公司。

由于深度学习的最新进展，研究人员正在利用深度学习来增强漏洞检测能力，并取得了令人期待的结果。早期模型如Devign和ReVeal依赖于图神经网络等架构，而更近期的最先进模型已经向基于Transformer的架构转变。CodeBERT利用了代码和注释的掩码语言模型（MLM）目标以及替换令牌检测目标进行预训练。GraphCodeBERT利用语义级别的代码信息，如数据流，来增强它们的预训练目标。最新的模型UniXcoder利用AST和评论等跨模态内容来丰富代码表示。

![image-20240425152244639](https://s2.loli.net/2024/04/25/RkLNTcthna74p3i.png)

然而，深度学习工具面临一个重要挑战，那就是模型在学习和使用代码特征与标签之间的虚假相关性时，没有使用根本原因来预测漏洞，而是使用了这些虚假相关性。我们将这些在虚假相关性中使用的特征称为虚假特征。例如，在图1a中，这段代码包含了一个内存泄漏。SOTA模型CodeBERT以非常高的置信度（概率=0.95）正确检测到了这个漏洞。然而，在我们重构了代码并且重命名了变量之后（图1b），模型预测该函数为非易受攻击的。在图2中，我们展示了变量名称的改变导致了其代码表示从易受攻击的簇移动到了非易受攻击的簇。



![image-20240425152329589](https://s2.loli.net/2024/04/25/NSl1gyEG9LaTCou.png)

显然，模型没有使用漏洞的原因——"分配的内存必须在每个路径上释放"来进行预测。在这个例子中，av_malloc和av_freep是因果特征——是对av_freep API的错误使用导致了漏洞。然而，模型将s、nbits和inverse与易受攻击的标签相关联，将out1、dst0和out0与非易受攻击的标签相关联。这种相关性是虚假的；那些变量名称是虚假特征。我们相信，虚假相关性是阻止模型具备鲁棒性和适用于未知项目的重要原因。

为了解决这个挑战，在本文中，我们引入了因果性（统计学的一个分支）到深度学习漏洞检测中。我们开发了CausalVul，并应用了一种因果学习算法，该算法实现了因果的做法和反向门控准则，旨在禁用模型使用虚假特征，并促使模型使用因果特征进行预测。

我们的方法由两个阶段组成。在第一阶段中，我们致力于发现模型使用的伪特征。由于深度学习模型难以理解的特性，这项任务具有挑战性。为了解决这个问题，我们设计了一种新颖的扰动方法，受到对抗鲁棒性测试的启发。这种扰动改变代码的词法标记，但保留代码的语义，就像编译器变换一样。我们假设模型可能会将变量名作为伪特征，如图1所示，并且模型可能还会使用API名称作为另一个伪特征。相应地，我们设计了PerturbVar和PerturbAPI这两种方法来改变程序，然后观察模型对这些程序的预测是否发生变化。通过我们的实证研究，我们验证了模型确实将某些变量名和API名称作为伪特征，并且对于易受攻击的和非易受攻击的示例，模型使用不同的这些名称集合（第3部分）。

为了防止模型使用这些伪特征，在第二阶段，我们应用了因果学习，这包括在现有深度学习模型之上的特殊训练和推理过程。因果学习对给定模型进行干预，计算如果没有伪特征时模型的行为。这可以通过训练数据的反向门标准以及我们对伪特征的了解来完成。具体来说，我们首先训练一个模型并明确地将已知伪特征 𝐹 编码到模型中。在推理时间为一个示例 𝑥 时，我们使用 x 的表示联合一组不同的伪特征，这样模型就无法使用 𝐹 来做出对 𝑥 的最终决策。

我们使用Devign [27]和Big-Vul [12]这两个真实世界的漏洞检测数据集评估了CausalVul，并研究了三种领先模型 — CodeBERT、GraphCodeBERT和UnixCoder。实验评估显示，CausalVul中的因果模型学会忽略伪特征，相较于SOTA，在Big-Vul数据集中漏洞检测的整体性能提高了6%，在Devign数据集中提高了7%。CausalVul还展现了在泛化测试上的显著提升，将Devign数据集训练模型的性能提高了高达100%，Big-Vul数据集训练模型的性能提高了高达200%。实验结果还表明，我们的CausalVul比当前的SOTA模型更稳健。在我们为鲁棒性测试构建的扰动数据上，它将性能提高高达62%在Devign上，100%在Big-Vul上。

总而言之，本文的贡献如下：

• 我们发现并在实验中证明当前深度学习漏洞检测模型中使用变量名和 API 名称作为虚假特征，

• 我们使用因果关系来制定深度学习漏洞检测，并应用因果深度学习来消除模型中的虚假特征，

• 我们实验性地证明，因果深度学习可以提高模型的准确性、稳健性和泛化能力。

## 方法以及概述

在图3中，我们展示了我们的方法CausalVul的概述。CausalVul包含两个阶段：（i）发现虚假特征和（ii）使用因果学习去除虚假特征。

![image-20240425152217481](https://s2.loli.net/2024/04/25/v359WHuROxUBqAQ.png)

*发现虚假特征。*首先，我们致力于发现深度学习漏洞检测模型中使用的虚假特征。虚假特征是输入数据集的属性，例如变量名，它们与目标标签（易受攻击/不易受攻击）存在关联，但并非是（非）易受攻击的根本原因，因此可能无法推广到所需的测试分布。为了发现虚假特征，在我们的方法中，我们首先根据我们的领域知识假设一个虚假特征。例如，我们假设变量名可能是漏洞检测任务的虚假特征——模型不应该根据名称做出决定。接下来，我们根据每个假设特征设计变化。具体来说，我们通过一种重构工具转换程序，改变特征，但不改变代码的语义（类似于编译器变换）。当我们观察到模型对许多这样的例子改变其预测时，我们得出结论，模型很可能依赖于该虚假特征。发现虚假特征是一项非常具有挑战性的任务。例如，随机更改变量名并不能揭示其虚假本质。我们需要仔细确定需要通过哪些名称更改哪些变量名，以最大程度地实现其影响。据我们所知，我们还没有看到关于在漏洞检测模型中发现虚假特征的系统研究。

*去除虚假特征。*一旦我们发现虚假特征，我们尝试在这一步减少这些特征对模型决策的影响。一个简单的方法是通过随机化虚假特征来增强训练数据，然后重新训练模型。然而，对于大规模的、预训练的、代码表示/深度模型来说，这将需要很长时间。因此，我们采取现有表示方法，应用因果学习，在推理时禁用模型使用虚假特征。这确保了模型主要依赖因果特征进行预测。

我们通过以下方式实现这个目标。首先，我们将现有的表示和待消除的已知虚假特征作为输入。然后，我们训练一个模型，使得在这一步学到的表示对目标虚假特征有着极高的偏好。接下来，在推断过程中，模型基于输入表示做出决策，同时忽略在前一步学到的偏好表示。具体而言，我们对包含不同虚假特征的一组示例进行边缘化处理，并防止推断过程利用目标虚假特征做出最终决策。这种方法依赖于"do-计算"和"后门准则"的原理，在第4和第5节中将分别详细解释。

据我们所知，这是首个将因果学习与深度学习模型用于漏洞检测的工作。它有助于研究人员了解因果关系在多大程度上可以帮助深度学习模型消除虚假特征，以及在消除虚假特征后我们能够在模型准确性、鲁棒性和泛化能力方面取得多大程度的改进。类似的因果学习方法已经应用于计算机视觉 [9]。一个关键的区别是，视觉域是连续的，即像素值是连续的实数，而程序特征是离散的。因此，在程序数据领域，我们可以明确地发现虚假特征，然后以有针对性的方式应用因果学习。

## 发现虚假特征

在这一部分中，我们通过保持语义不变的扰动技术展示了发现虚假特征的方法。

虚假特征是指与输出产生虚假关联的特征。这些虚假的关联通常只存在于训练数据中，而不在测试数据中，尤其是在扰动数据或未知项目下的测试数据中。这是因为虚假特征与因果关系不同，在不同领域中会发生变化，并且不再对预测有用，如图1所示。因此，确定某个特征是否为虚假特征的一种常见方式是扰动该特征的值，并观察输出的变化情况。

根据这个想法，我们最初假设了两个虚假特征，即变量名和API名称，当前的深度学习模型可能用于漏洞检测。我们选择这两个虚假特征作为概念验证，以检查如果我们分别扰动词法和句法特征，同时保持程序的整体语义不变，模型的预测是否会改变。我们应用了一个现有的代码重构工具NatGen [3]来扰动测试数据中的代码，并观察原始测试集和扰动测试集之间的模型性能是否变化（详见第3.1节中的问题阐释）。有趣的是，我们发现随机更改变量名和函数名并不会降低模型的性能。因此，我们设计了能够展示这两个虚假特征的新型扰动方法。请参阅第3.2至3.4节。

### 问题的提出

给定一个扰动𝑝，一个代码样本𝑠，和一个已训练的模型𝑀，如果满足以下条件，我们将发现一个虚假特征：(1) 应用扰动𝑝(𝑠)不改变函数的语义，(2) 在转换之后模型的预测发生改变，(3) 扰动𝑝的候选项来自训练分布。
我们将使用F1分数作为对于我们不平衡数据集和条件数(2)的更全面的评估指标。降低的F1分数表明应用扰动𝑝(𝑠)导致了错误分类（标签翻转）。

### PerturbVar： 作为虚假特征的变量名

为了证明有些变量名实际上被模型用作虚假特征，我们设计了一个极其扰动的测试集。我们分析训练数据集，根据它们在易受攻击函数和非易受攻击函数中出现的频率对变量名进行排序。在替换测试集中现有变量名时，我们随机从相反类标签的前K个最频繁出现的变量名中选取一个：对于一个非易受攻击样本，我们把现有的变量名替换为来自易受攻击训练集的变量名，但是该变量名不在非易受攻击训练集中出现，反之亦然。我们对测试集中的每个样本都应用这种扰动。

![image-20240425152939013](https://s2.loli.net/2024/04/25/Cm9d5TlRDk4Kfab.png)

**观察结果** 如表 1 所示，在 Devign 数据集上，我们的 F1 分数降低了 11.5%，在 Big-Vul 数据集上降低了 20%。我们在所有数据集和多种架构上都观察到了性能下降的情况： CodeBERT、GraphCodeBERT 和 UniXcoder。不过，在随机化设置中，性能与基线相比几乎没有变化。在非漏洞代码样本中引入常见的漏洞名称会导致模型将样本错误地分类为漏洞。相反，在易受攻击的代码样本中引入常见的非易受攻击名称会导致模型将样本错误地分类为非易受攻击。使用的变量名越常见（即 Top-K 越低），性能下降得越厉害。

### PerturbAPI：作为虚假特征的 API 名称

现代程序经常使用API调用。我们推测模型可能会建立API名称与漏洞之间的虚假相关性。类似于PerturbVar中使用的方法，我们分别对训练数据中的易受攻击示例和非易受攻击示例中API调用的频率进行排名。然后，我们将API调用（在非易受攻击示例中排在前100位且在易受攻击示例中并不频繁出现的调用）插入易受攻击示例，反之亦然。为了保留代码的语义，我们将这些API调用插入为“死代码”，即这些代码永远不会被执行。
我们在代码样本中的𝑛个随机位置注入死代码。死代码块由𝑚个不同的函数/API调用组成（𝑚和𝑛是可配置的，我们使用𝑚 = 5，𝑛 = 5来生成表2中的结果）。我们用一个不满足的条件来保护这个API调用块，确保该循环永远不会被执行，如图4所示：



```c
while ( _i_4 > _i_4 ) { 
    
	tcg_out_r(s , args[1]); help_cmd(argv[0]); 
    
 	cris_alu(dc , CC_OP_BOUND , cpu_R[dc -> op2] , cpu_R[dc -> op2 ] , l0 , 4);
  	
    RET_STOP(ctx); tcg_out8(s , args[3]); 
}
```



![image-20240425153251565](https://s2.loli.net/2024/04/25/p6aHYS8UouMCkVr.png)

![image-20240425153307594](https://s2.loli.net/2024/04/25/sVtLx8N7vap1E5e.png)

**观察**。我们将我们的结果与（1）基准性能和（2）随机死代码转换性能进行比较，如表2所示。我们的结果表明，由API调用组成的死代码相对于基准平台和随机死代码转换会严重影响模型性能。随着API调用数量和注入位置的增加，模型性能会相应地降低。在Devign上，基准模型性能下降了多达28.73％，在Big-Vul上下降了27.99％。

### PerturbJoint： 将它们结合在一起

我们假设两个虚假特征的组合将进一步降低模型的性能。我们建立了这项研究，对于第2节PerturbAPI数据集中构建的每个样本，我们将现有的变量名称替换为对立类别中排名前K的最频繁变量名称的随机选择（使用第1节中相同的方法）。

**观察**。在表3中，我们的结果显示API死代码和变量重命名的组合进一步降低了模型的性能。当应用设置的组合时，模型的退化更为严重。在综合设置中，Devign的性能降低了41.37%，Big-Vul降低了33.89%。

**总结**。在这一部分，我们调查了多个数据集和多个SOTA模型，揭示了变量名称和API名称是这些模型经常利用的虚假特征。有趣的是，模型将不同的变量名称和API名称视为不同标签的虚假特征。因此，显而易见的是，只有经过精心设计的扰动，而不是随机扰动，才展示了这些虚假特征的使用，从而导致了模型在扰动数据集上的性能下降。

![image-20240425153810414](https://s2.loli.net/2024/04/25/uKb1YjHTaBW357F.png)

## 因果学习以去除虚假特征

在本节中，我们将介绍如何应用因果学习来去除脆弱性模型中的虚假特征。

### 漏洞检测因果图

应用因果学习，我们的第一步是构建一个因果图，从统计学的角度模拟漏洞数据是如何生成的，如图5(a)所示。因果图直观地表示不同变量或事件之间的因果关系。在图中，节点表示随机变量，节点之间的有向边表示因果关系。因此，A→B表示变量A直接影响变量B。两个节点之间没有边意味着它们之间没有直接的因果关系。在图5(a)中，𝑋表示程序中的一个函数。代码中是否存在漏洞直接取决于该函数。因此，我们从𝑋到𝑌之间添加一条边，其中𝑌是漏洞检测的标签，1表示存在漏洞，0表示不存在漏洞。

![image-20240425153920131](https://s2.loli.net/2024/04/25/dmXBuPh8wYIRi7H.png)

在因果图中，我们使用 𝐸，即环境，来模拟生成数据集的领域。例如，在训练时，即 𝐸 = 0，此环境表示例如代码由某些开发人员编写，面向某些软件应用。而在模型的测试或部署时，即 𝐸 = 1，代码可能会连接到不同的开发人员和应用程序。这种与环境相关的因素使用 𝑈 进行建模。它是一个潜在变量，在训练数据分布中并不直接观察到。例如，𝑈 可以表示程序员的专业知识或编码风格，或者软件应用的类型。由于 𝑈 的存在，代码中可能会存在不同的伪特征，就像我们在第3节中展示的两种，分别表示为 𝑀1 和 𝑀2。

𝑀1 代表变量命名风格，因为开发人员可能喜欢在他们的代码中使用某些格式的变量命名。同样，𝑀2 代表 API 名称，因为某些开发人员或应用程序可能更可能使用一组特定的 API 调用。这些关于 𝑀1 和 𝑀2 的基本（文本）特征影响着代码。因此，我们将边缘连接到 𝑀1 和 𝑀2，分别表示为 𝑋。需要注意的是，因果图可以扩展以整合更多此类伪特征。我们会将此留给未来的工作来解决。

在图5（a）中，我们还有边缘连接 𝑈 到 𝑌，表示 𝑈 可以影响到 𝑌。例如，初级开发人员更可能引入漏洞；类似地，某些 API 更可能引入漏洞，比如 SQL 注入。

### 应用因果关系

当我们训练一个深度学习模型，使用代码 𝑋 作为输入，来预测漏洞标签 𝑌 时，我们会学习在训练数据集中的相关性 𝑃(𝑌|𝑋)。然而，当我们将这个训练好的模型部署到一个新环境中，比如处理扰动数据集或者未知数据集时，领域会发生变化。由于从 𝐸 为 0 到 1 的转变，相关性 𝑃(𝑌|𝑋) 也会不同，表示为 𝑃(𝑌|𝑋, 𝐸 = 0) ≠ 𝑃(𝑌|𝑋, 𝐸 = 1)。从因果图的角度来看，𝑋 和 𝑌 都是 𝐸 的后裔，会受到其变化的影响。

为了提高模型在超出训练环境的新环境中的泛化性能，我们的目标是学习在新环境和领域中不变的信号。通过采用因果学习，我们对因果图进行干预，使得𝑋和𝑌之间的相关性无论环境𝐸如何变化都保持不变。在因果关系中，这种干预是通过do-计算实现的，记作𝑃(𝑌 |𝑑𝑜(𝑋 = 𝑥))。

关键思想是，我们假设有一个能够编写完美且无偏见的代码 𝑥 的神谕；然后我们使用这样的代码来替换原始代码 𝑋 以检测漏洞。这样一来，我们通过移除所有指向节点 𝑋 的入边并将新代码放置在这里 𝑋 = 𝑥 来创建一个新的因果图。参见图5（b）。在这个新的因果图中，𝐸 只能影响 𝑌，而不是 𝑋，因此 𝑋 和 𝑌 的相关性将不再依赖于 𝐸。换句话说，在这个新的因果图中，𝑥 和 𝑌 的相关性将成为跨不同领域的不变信号，例如从训练到测试。当我们在这个新的因果图中学习相关性时，学到的模型可以推广到新环境，因为这种关系在测试中也成立。在接下来的部分中，我们将解释如何从观察数据中计算出 𝑃(𝑌 | 𝑑𝑜(𝑋 = 𝑥)。

### 通过观察数据估计因果关系

图5(a)从数据生成的角度建模了漏洞数据的联合分布。图5(b)在对𝑋进行干预后呈现了因果图，使我们能够在不同领域中进行推广。挑战在于进行干预并获得完美、无偏的代码几乎是不可能的。为了在不实际进行干预的情况下估计因果效应，我们将反门标准[21]应用于给定的𝑋和𝑌的观测数据，以推导出因果效应𝑃(𝑌 |𝑑𝑜(𝑋))。

背门准则教我们如何在因果图中阻止所有非因果路径从𝑋到𝑌，以便我们可以计算因果效应。例如，在图5（a）中，节点𝑈可以导致𝑋，节点𝑈也可以导致𝑌；因此，在观察数据中可能存在𝑋和𝑌之间的相关性。然而，这种相关性不一定表示从𝑋到𝑌的因果关系。例如，初级开发人员可能喜欢用变量名myvar编写代码，并且初级开发人员更有可能在他们的代码中引入漏洞。在这种情况下，我们经常会看到数据集中检测到带有myvar的代码存在漏洞。然而，myvar并不是漏洞的原因。myvar和漏洞之间的相关性是错误的。为了消除这种虚假相关性，背门准则指出我们需要将𝑋条件化为𝑀1和𝑀2；结果，我们可以删除指向𝑋的入边，从而影响从𝑈到𝑋的因果关系。数学上，我们将得到图5（b）的以下公式。
$$
P(Y|do(X=x))=\sum_{M_1,M_2}P(Y|X=x,M_1,M_2)P(M_1)P(M_2)
$$
为了计算式（1），我们采用了来自文献[9]的算法。首先，我们训练一个模型，用于从𝑅和𝑀（为简单起见，我们将𝑀1和𝑀2表示为𝑀）预测𝑌，记为𝑃(𝑌 | 𝑅, 𝑀)，其中𝑅是由已有深度学习模型计算得到的𝑋的表示，而𝑀是编码杂音特征的𝑋的表示。例如，如果我们将𝑋的每个标记编码到𝑀中，那么我们将在𝑀中有变量名和API名称的编码。由于𝑅是模型学习的表示，该算法假设𝑅同时编码了因果特征和杂音特征。将𝑅和𝑀共同用于训练的目标是特别在模型𝑃(𝑌 | 𝑅, 𝑀)的𝑀组件中编码目标杂音特征。

在推理过程中，我们使用模型 𝑃(𝑌 | 𝑅, 𝑀) 通过等式 (1) 计算通过给定 𝑋 = 𝑥 的数据 𝑃(𝑌 | 𝑑𝑜(𝑋 = 𝑥))。反向门准则指示我们首先从包含不同虚假特征的 𝑋 中随机抽取示例。然后，我们通过加权平均对虚假特征进行边缘化，即，$\sum_{M_1,M_2}P(Y|X=x,M_1,M_2)P(M_1)P(M_2)$。这可以使用上述已训练模型 𝑃(𝑌 | 𝑅, 𝑀) 计算。直觉上，数据中的虚假特征被加权平均抵消掉了（详见[22]为什么会这样）。模型将根据剩余的信号进行预测，这些信号是留在 𝑅 中的因果特征。

## 因果漏洞检测算法

在本节中，我们将介绍用于计算方程（1）项的算法。在算法1中，我们将训练一个𝑃(𝑌|𝑅, 𝑀)模型。在算法2中，我们展示了如何应用𝑃(𝑌|𝑅, 𝑀)并使用倒门准则来在推理过程中消除虚假特征。

### 训练P(Y|R,M)

算法1接受训练数据集 𝐷 和我们打算移除的虚假特征，即目标虚假特征 𝑡，作为输入。例如，这些可能是变量名和/或 API 名称，正如第3部分所述。

算法1的目标是从训练数据中学习 𝑃 (𝑌 |𝑅, 𝑀)。我们使用现有深度学习模型计算的原始输入 𝑥 的嵌入（𝑟）。在第4行，训练将迭代 𝑛 个周期。对于每个标记数据点（𝑥, 𝑦），我们首先在第6行获取 𝑥 的 𝑟。

在第7行，我们选择与 𝑥 共享目标虚假特征 𝑡 但在根本原因 𝑟 上有所不同的 x'。这意味着（1）𝑥' 应该与 𝑥 具有相同的标签；如第3部分所示，虚假特征是与标签相关的；（2）𝑥' 应该与 𝑥 共享 𝑡，这样模型更有可能在 𝑀 部分编码𝑡；以及（3）𝑥' 不应该是 𝑥，这样在第9行的训练中，模型会依赖于 𝑀𝑥' 中的虚假特征而不是 𝑟 进行预测。我们最终的目标是在模型中编码 𝑥 的根本原因在 𝑅 部分，并在 𝑀 部分中共享 𝑥 和 𝑥' 的目标虚假特征 𝑡。

在表4中，我们设计了不同的方式来选择 x' 在第7行，设置 Var1-Var2 用于移除变量名的虚假特征，设置 API1-API3 用于 API 名称，设置 Var+API 用于变量和 API 名称的情况。

第一个想法是从训练数据集中选择 x'，使其与 x 共享最多的虚假变量或 API 名称。参见 Var1 和 API1 的行。第二个想法是构建一个示例 x'，使其与 x 共享 𝑘 个变量/API 名称。参见 Var2 和 API2 的行。类似地，通过设置 API3 构建 x'，使其具有前 𝑘 个虚假 API 名称。我们还尝试了这种方法来处理变量名，但结果不理想。在设置 Var+API 中用于移除两类虚假特征时，我们从 Var1 设置中选取 x'，然后使用 API3 设置进行转换。与其他组合相比，在我们的实验中，这种方法取得了最佳结果。

![image-20240425160718640](https://s2.loli.net/2024/04/25/L7byWATZz6olJSh.png)

### 在推理中取消虚假特征

在算法2中，我们解释我们的推理过程。要为函数𝑥预测标签，我们首先在第2行提取𝑟。现有的方法直接使用𝑟来预测输出𝑦。由于𝑟中既有虚假特征又有核心特征，它们都将被用来预测脆弱性。

![image-20240425160824209](https://s2.loli.net/2024/04/25/jVs6OMX5Qr4q3fe.png)

在这里，我们将使用我们的因果算法在推理中去除虚假特征。关键想法是通过对所有种类虚假特征的预测进行平均化，以抵消虚假特征的贡献。为此，在第4行，我们从训练数据集D中随机抽取不同的x'K次。在第8行，我们计算方程（1）。我们假设了𝑃(𝑀′𝑥)的均匀分布。最后，在第9行，我们做出预测。𝑎𝑟𝑔𝑚𝑎𝑥_𝑦表示从脆弱/非脆弱类别中选择具有更好概率的标签。

![image-20240425160841589](https://s2.loli.net/2024/04/25/ycVtH3TxONdiKvZ.png)

## 实验设置

### 实现

我们使用 PyTorch 2.0.13 版本，搭配 Cuda 12.1 和 transformers4 库来实现我们的方法。所有模型都在单个 NVIDIA RTX A6000 GPU、Intel(R) Xeon(R) W-2255 CPU 和 64GB 内存上进行微调。Transformer 模型的预训练权重和分词器是从原始作者提供的链接中获取的。我们使用了 Adam 优化器来微调我们的模型。在设置批量大小为 32 的情况下，模型训练了 10 个 epochs。学习率被设置为 2e-5。我们使用我们的训练数据来训练模型，并根据与验证集的 F1 分数选择最佳微调权重。在测试集评估期间，我们使用这个权重。我们把 K 设为 40（算法2，第3行），因为这个值在我们尝试的数值中表现最好。

### 数据集以及模型

我们考虑了两个漏洞检测数据集：Devign [27] 和 Big-Vul [12]。Devign 是一个平衡的数据集，包括了27,318个例子（53.22%的有漏洞），这些例子是从两个不同的基于大型C编程的项目 Qemu 和 FFmpeg 中收集而来的。由于 Zhou 等人 [27] 没有提供训练、测试和验证集的划分，我们使用了 CodeXGLUE 作者发布的划分。Big-Vul 数据集是一个不平衡的数据集，包括了188,636个例子（5.78%的有漏洞），这些例子是通过在公共漏洞和暴露（CVE）数据库中进行爬取收集而来的。对于这个数据集，我们采用了 LineVul 作者发布的划分。

我们评估了三个最先进的模型 CodeBERT、GraphCodeBERT 和 UniXcoder 作为算法1中的模型 𝑃 (𝑅|𝑋 )。表示 R 是从这些模型的最后一个隐藏层的输出嵌入中提取的。为构建网络$ \hat 𝑃 (𝑌 |𝑅, 𝑀)$，首先，我们通过这些变压器模型的前四个编码器块来传递 𝑥’，获得的输出嵌入视为 M。我们使用了这些变换器模型中的前四个编码器块（根据实验结果，这是我们发现的最佳层）来计算 M，因为较早的层学习底层特征，而伪特征倾向于是底层特征。然后，M 与 𝑅 进行拼接。最后，使用一个具有两层全连接网络来预测 𝑌。

## 7 评估

 我们研究了以下研究问题：

RQ1：CausalVul能否提高模型的准确性？

RQ2：CausalVul能否提高模型的鲁棒性和泛化能力？

RQ3：（消融研究）不同的设计选择如何影响CausalVul的性能？

### 7.1 RQ1: 模型准确性实验设计。

要回答这个研究问题，我们将我们的因果推理方法应用于三种最先进的基于Transformer的漏洞检测模型 - CodeBERT、GraphCodeBERT和UniXcoder。我们将这些模型的默认版本（即没有因果关系的版本）作为基线。我们将这些默认版本称为基本模型。

![image-20240425161531869](https://s2.loli.net/2024/04/25/o6tuMbaZjkhAlvg.png)

我们在表4中展示的所有因果设置上进行了实验，即Var1和Var2，API1、API2和API3，以及Var+API。我们在同一份未经扰动的原始测试集上评估了基本模型和因果模型，并使用了F1度量标准。我们使用不同的随机种子训练所有模型三次，将平均F1分数作为我们的最终分数（对于所有研究问题都是这样做的）。

结果：表5显示了结果。在所有设置、所有数据集和所有模型中，CausalVul都优于基本模型。对于Devign数据，相较于CodeBERT基本模型，CausalVul在F1分数上的Var1、API3和VAR+API分别提高了4、5和4个百分点。在UniXcoder模型中，这三种方法的提高分别为2、4和2个百分点。使用GraphCodeBERT模型时，我们的API3和Var+API方法相较于基本模型分别提高了2和3个百分点。总的来说，我们的方法在F1分数上相较于基本模型提高了2-5个百分点。对于Big-Vul数据集，我们使用CodeBERT模型的因果方法表现出了2-3个百分点的提高，GraphCodeBERT和UniXcoder模型也是如此。

据我们所知，这些是这些广泛研究数据集中报告的最佳漏洞检测结果。有人可能认为，在忽略虚假特征的同时，模型可能会降低一些内分布准确性，因为虚假特征也在正常情况下提供一定帮助。然而，令人惊讶的是，我们发现CausalVul正在学习更重要的因果信号，这种信号弥补了虚假特征的缺失，并提高了内分布准确性。

**结果：RQ1。CausalVul表现优于其他预训练模型，表明因果学习专注于漏洞的根本原因，通过学习忽略虚假特征。整体上，我们的因果设定在Devign数据集中的F1上有高达6%的改进，Big-Vul数据集中的F1上有7%的改进。**

### 7.2 RQ2：模型鲁棒性和泛化实验设计。

对于鲁棒性评估，我们比较了因果模型与基准模型在第3节中呈现的三个扰动数据集上的表现。Var1和Var2在扰动变量数据集上运行，根据表1，在相应的基准模型上表现最差。例如，扰动变量数据集通过Top 5和Top 10最常见的变量名对vanilla CodeBERT和GraphCodeBERT模型的性能影响最大。因此，我们选择Top 5与CodeBERT进行比较，选择Top 10与GraphCodeBERT进行比较。同样，API1-API3在最差的扰动API数据集上运行，Var+API在相应模型的最差扰动联合数据集上运行。

为了研究模型的泛化性能，我们评估了在Devign数据集上训练的模型在BigVul测试数据集上的表现（排除了重叠项目FFMPEG）。同样，我们在Big-Vul上进行训练，并在Devign数据集上进行测试。对于这两个实验，我们在表4中尝试了所有设置，并使用F1作为度量标准。

鲁棒性结果：表6展示了三个区块中的鲁棒性表现：上区块展示了使用最差扰动变量数据时的结果，中区块和下区块分别展示了使用最差扰动API和最差扰动联合数据时的结果。

![image-20240425161837472](https://s2.loli.net/2024/04/25/EMhfxeVlWgsRb5p.png)

在Var1和Var2之间，Var1在Devign数据上表现更佳，F1值分别比CodeBERT、GraphCodeBERT和UniXcoder模型提高了6、3和2个百分点。对于Big-Vul数据，Var1在CodeBERT模型上表现更好，提高了1个百分点，而Var2在其他两个模型上分别提高了4个和3个百分点。总体而言，Var1和Var2方法在鲁棒性方面均优于基准模型。在API1、API2和API3之间，API3在Devign数据中表现更好，分别在三个模型上提高了10、5和22个百分点。对于Big-Vul数据，API2在CodeBERT模型上表现更好，比CodeBERT基准提高了3个百分点。在GraphCodeBERT中，API1和API2表现类似，均提高了2个百分点。VAR+API设置显示出与基准性能相似的改进趋势。

在图6中，我们展示了对Devign易受攻击数据的预测概率密度。在每个子图中，X轴表示易受攻击的预测概率，Y轴表示以该概率进行预测的样本数。橙色线绘制了因果模型，蓝色线绘制了普通模型。图表明，易受攻击数据的整体预测概率增加，这意味着模型在预测漏洞时更有信心。实验结果表明，普通模型和因果方法之间的概率密度总体差异在统计上是显著的，P值 <<< 0.05，且具有不同的效应大小，详见表7。

![image-20240425161852644](https://s2.loli.net/2024/04/25/6zWjP9L2NZ8ClSg.png)

![image-20240425161941235](https://s2.loli.net/2024/04/25/fJTivoRlhQ4stp5.png)

我们还调查了有多少例稳健数据在普通模型（如图1所示）中被错误预测，但在CausalVul中被正确预测。表8显示，CausalVul在这方面成功地预测了大量普通模型中错误预测的数据。

![image-20240425162009701](https://s2.loli.net/2024/04/25/Rg83N64IxlrPMmK.png)

广义化结果：我们在表9中展示了广义化结果。Devign列定义了在用Devign训练集训练的模型对Big-Vul测试集进行评估时的F1分数。同样，Big-Vul列表示在用Big-Vul训练集训练的模型对Devign测试集进行评估时的F1分数。我们的结果表明，当模型在Devign训练集上训练，将Big-Vul测试集用作外部分布（OOD）时，我们的因果方法在CodeBERT和UniXcoder模型中显示出12个百分点的改进。但是，当将Devign测试集作为Big-Vul数据训练的模型的OOD数据时，CodeBERT模型显示出7-10个百分点的改进，GraphCodeBERT模型最多显示4个百分点的改进，而UniXcoder模型显示出6-14个百分点的改进。

![image-20240425162110148](https://s2.loli.net/2024/04/25/5Szo7wYWErOTkhQ.png)

在这个研究问题中，我们展示了因果学习具有显著提高鲁棒准确性和泛化能力的潜力。在这种情况下，由于评估数据中可能不存在虚假特征，学会忽略它们有助于显著改善 CausalVul 的性能。
结果：RQ2。CausalVul 在 Devign 和 Big-Vul 的鲁棒性数据中分别显示了高达 62% 和 100% 的改善。CausalVul 对两个数据集的泛化性能分别提高了高达 100% 和 200%。

### 7.3 RQ3：消融研究实验设计。

在这个研究问题中，我们调查了CausalVul的设计选择。在第一个设置中，我们使用K=1，并在算法2中将𝑥'设置为𝑥。在这里，我们研究了如果不使用边缘化，我们的方法的性能如何。我们在鲁棒性测试数据上评估了模型。由于篇幅限制，我们只展示了Var+API设置的结果。

![image-20240425162555684](https://s2.loli.net/2024/04/25/83acJ1PUkGEF7vV.png)

在第二个设置中，我们研究了在使用不同的早期层来表示𝑀时我们的方法的性能。因此，我们从第一、第二、第三和第四层提取𝑀，并将其用于我们的因果方法。

结果。在图7中，我们使用与图6类似的概率密度图。橙色线表示K=40，蓝色线表示K=1。从结果来看，我们可以清楚地看到，对于易受攻击和非易受攻击标签，K=40学习得更好，并且对于真实标签报告了更自信的预测。

![image-20240425162607890](https://s2.loli.net/2024/04/25/ZY15tRiNeyj7fAO.png)

表格 10 展示了使用不同早期层提取 𝑀 的结果。我们选择了所有模型使用 Var+API 配置来展示结果。对于所有模型和数据集，第四个层次报告了最佳性能。

结果：RQ3. 我们的结果表明，边际化（反门准则）有助于模型专注于因果特征而非虚假特征。另一方面，我们发现第四层比前三层更适合用于从 𝑥′ 中计算 𝑀。

## 有效性威胁

为了发现虚假特征，我们的实验设计遵循文献[3]，并确保我们的干扰遵循一致性、自然性和语义保留。

由于更好的随机种子，深度学习模型可能会报告改进的结果。我们所有的实验都是使用三个随机种子运行的。我们的因果模型在所有数据集和所有模型上都一直表现出改进。我们进行了统计测试，以显示我们的改进具有统计显著性。除了F1之外，在第6和第7图中显示的概率密度图也强烈证明了我们的改进。

因果学习假设代码表示𝑅学习了因果特征。尽管我们不确定是否是这种情况，但我们在所有设置中都看到了结果的改进。

我们的评估涉及两个真实世界的漏洞数据集，包括平衡和不平衡数据，以及三种SOTA模型。在未来，我们计划尝试更多的数据集和模型。

## 相关工作领域

深度学习在漏洞方面的应用：深度学习漏洞模型可以分为基于图神经网络（GNN）或基于变压器的模型。基于GNN的模型将AST、控制流和数据流信息捕捉到图形表示中。

最近的基于GNN的模型[6, 7, 17, 25, 27]提出了基于语句级别的漏洞预测。相比之下，变压器模型是在自监督学习设置下进行预训练的。它们可以根据三种不同设计进行分类：基于编码器、基于解码器[8]和混合架构[3, 4, 15]，这些架构结合了两种方法的元素。

例如，CodeBERT[13, 16]等基于编码器的模型通常采用掩码语言模型（MLM）的预训练目标；有些结合对比学习方法[5, 10]，而其他则旨在使预训练更加执行感知[11]、二模态[4]或自然化[3]。漏洞检测一直是这些模型的重要下游任务之一。在本文中，我们使用了三种最新的变压器基模型：GraphCodeBERT、CodeBERT和UniXcoder，并展示因果性可以进一步提高它们的性能。

针对这些模型的鲁棒性和泛化性也有研究。最近的研究，如Steenhoek等人[23]，评估了几个最新技术的漏洞模型，以评估它们对未知数据的能力。此外，他们涉及到虚假特征，并发现一些令牌，如“error”和“printf”，经常被用于进行预测，这可能导致错误的预测。

在另一项关于基于深度学习的漏洞检测的系统性调查中，Chakraborty等人[7]指出，漏洞检测模型没有捕捉到相关代码特征，而是依赖于训练分布中的不相关方面（比如特定的变量）来做出预测。我们的工作设计了新颖的扰动，以验证假设的虚假特征，发现不同的名称被用于不同的标签作为虚假特征。

在现有研究中，还没有进行过这种研究。软件工程中的因果学习：据我们所知，将因果关系应用于软件工程是相对较新的。Cito等人[9]是最相关的一项最新研究，他们调查了对源代码进行扰动会导致模型“改变主意”的问题。这种方法利用掩码语言模型（MLM）生成一个“自然”扰动的搜索空间，可以改变模型的预测。这些自然扰动称为“反事实解释”。我们的工作也寻求自然扰动，并使用程序中的变量名和API名称来执行扰动。

但是，我们的工作不同之处在于，我们要求我们的扰动保持语义一致。此外，我们促使模型改变决策的目标是发现虚假特征，而不是解释bug的原因。

在其他领域的因果学习上也存在一些工作：我们的工作从毛氏等人那里得到了启示。这项工作通过因果学习来应对视觉领域的鲁棒性和泛化性。他们将“水鸟”和“陆鸟”作为两个领域，并通过应用反事实解释，因果模型可以学习到这两个领域中鸟类的“不变性”，从而实现更好的泛化性。他们的工作没有发现虚假特征，他们的因果学习也没有针对虚假特征。

## 结论与未来工作 

本文提出了检测因果性脆弱性的第一步。我们解决了深度学习脆弱性检测的几个重要挑战。首先，我们设计了新颖的扰动，以暴露深度学习模型用于预测的虚假特征。其次，我们使用因果关系和因果推断来形式化深度学习脆弱性检测问题，以便我们可以应用因果学习算法去除虚假特征，并推动模型使用更加健壮的特征。我们设计了全面的实验并证明了 CausalVul 改善了脆弱性检测的准确性、鲁棒性和泛化能力。在未来，我们将计划发现更多的虚假特征，并探索因果学习在软件工程中的其他应用。