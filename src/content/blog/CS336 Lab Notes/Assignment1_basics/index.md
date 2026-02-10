---
title: 'CS336 Lab Notes 1 Basics of a mini LLM' # (Required, max 60)
description: '忽然落下的夜晚，灯火如隔世般阑珊。昨天已经去得很远，我的窗前已模糊一片。——朴树《且听风吟》' # (Required, 10 to 160)
publishDate: '2026-01-10 16:30:00' # (Required, Date)
tags:
  - CS336 Lab Notes
heroImage: { src: './f53d0d75af8e0900c3778a29a25ffe44.jpg', alt: 'Violet', color: '#B4C6DA' }
draft: false # (set true will only show in development)
language: 'Chinese' # (String as you like)
comment: true # (set false will disable comment, even if you've enabled it in site-config)
---

# CS336 Lab1 Basics 实现基本思路与简洁代码

可以在[这个仓库](https://github.com/TRpuppy/CS336_TRpuppy/tree/master)找到我的代码实现。

## 实现BPE Tokenizer：Training与Encode, Decode

一个BPE Tokenizer，简单地说起来，就是两个规则表：

+ 一个从id到bytes序列的词汇表字典(vocabulary)。
+ 一个merge规则表，表示按顺序产生的merge rules.

但是大数据量会让一切看起来简单的东西变得不简单。我们的数据约有10G的文本，所以需要引入一些高效的处理，使得整个实现在内存和时间上都可接受。

### BPE Training

先约定一下：在这一节，我会把Pre tokenize后的单个结果叫成是"token"，BPE的词汇叫成是"word"。word的初始值就是所有utf8的bytes。

训练时，我们会每次寻找出现次数最多的相邻words对，然后合并这两个words成为一个新的word。

也是说起来简单，做起来不容易的一件事情。

#### PreTokenizer

我们先使用PreTokenizer，将文件切分为单词的形式，我们统计words对的时候，忽略跨Pre Token单词间的words对。按照指导文件，我们使用下面这个PAT正则表达式，在文件中找到所有匹配项。

```python
PAT = r"""'(?:[sdmt]|ll|ve|re)| ?\p{L}+| ?\p{N}+| ?[^\s\p{L}\p{N}]+|\s+(?!\S)|\s+"""
```

最终我把pretokenize后的结果，处理成一个字典`appear_times:dict[str:int]`的形式。`appear_times[word]`是一个整数，表示这个word在文件中出现了多少次。然后我们就只需要对整个文件过一次PreTokenizer，然后剩下的事情全部在`appear_times`这个词汇表里面办。

然而对于10G这个量级，一次遍历也是挺慢的，并且它对内存也不是一件很友好的事情。所以我使用了并行化设计，将整个文章切成若干个大小为1M的块(切点在Special Tokens处)，交给64个workers并行化处理(这里赞美一下我们伟大的180核CPU服务器)，每个块单独形成一个字典，最终再将整个字典合并。经过测试，这样并行化Pre Tokenize一个10G的文件大约只需要1分钟。

最终结果：总共发现 6601892 个唯一token, 总出现次数: 2,471,753,092，运行时间67s。(owt_train.txt)。

有了思路后，具体的代码交给AI完成。

#### Training Progress

按我们上面所说的，PreTokenize之后，剩下的事情我们只需要在PreTokenize后的token表上面办。这大大减小了我们的处理任务量。我们先会把每个token处理成一个token list，然后每次训练简单地分为两个步骤：

(1)统计所有word pair的出现频率，找出现频率最高的一个pair。

(2)对这个pair进行合并，更新所有token的word list。

最简单的方法，就是每次遍历两遍所有token的word list。但是其实总token数目N=6600000也不是一个太小的数目。如果我们需要生成一个words量为M=32000的词汇表。迭代M次肯定是不免要做的。如果每次遍历一遍所有Token，也即是每次N*Token_length的复杂度。如果Token_length认为是5的话，总复杂度就是10^12量级。且训练的过程也不好并行化。这是挺不能接受的一个方法。

我们最好寻找一点优化的方法。第一个想法就是其实word pair的出现频率你每次去找的话肯定浪费，因为它更新其实没多少。记录下来维护一个变化量其实是更好的办法。而这个变化量当然也好维护：你每次不是要更新word list么，更新word list的时候顺手把前后的word pair出现次数，消除的该减减，出现的该加加就好了。这样就有了一个统一的`appear_times:dict[(int,int):int]`字典，键是word pair，值是出现次数。

但是这其实并没有解决数量级问题，因为你每次更新的时候还是要遍历一遍所有token。我们能不能引入一个结构，描述每个word pair出现在哪些token里面，从而不去遍历一遍所有token，只针对性地找这个pair出现过的token呢？想了一下确实是可以的。

最开始我的实现是`appear_index:dict[(int,int):List[int]]`，键是word pair，值是一个token下标的列表，表示在哪些token里面出现过。这样，每次更新的时候，就只需要在这个列表对应的token里面处理就好了。

但是这有一个小问题，我举个例子：例如`he`这个word pair和`there`这个token。在初始化时会认为he出现在there中。但是如果我们第一次合并的是`th`这个word pair，那`he`这个word pair就不再出现了。虽然这不是什么大毛病：至少这样维护出来的列表总是保证出现word pair的token一定在列表中。

但是如果想做得更好，可以更新成`appear_index:dict[(int,int):Dict[int:int]]`。第一个键是键是word pair，第二个键是token下标，值表示这个word pair在这个token中出现过多少次。如果减到0了，就删除这个键。这也就是我最终的实现。

整体速度：训练阶段约30mins。(owt_train.txt，32000 vocabs)

### BPE Encoding and Decoding

下面我们切换一下语言约定：BPE的词汇表中的词汇统一称作token(在上面，它被称作word)，token对应的标号就叫token_id。

Decoding很简单，从token_id复原回token，再从token连接起来的bytes复原回UTF-8就可以了。同时目前为止，我们也没有大规模Decoding的需求，所以暂时可以不作任何优化。

Encoding基本的idea就是对输入的字符串按顺序应用merge rules。同时有一个细节需要注意：为了保持与训练时一致，我们在encoding时也需要使用相同的方式进行Pre Tokenize，然后忽略所有跨Pre_token的合并。所以我们的实现中，最小单位的函数设计成：`apply_merges(token:bytes)->list[int]`，这个函数只接受PreTokenize后得到的一个小token，返回最终的encode结果。

这个函数的具体实现：别傻傻地去遍历一遍所有merge rules(约32000个)了~~(没错说得就是我自己)~~，一个token大小约为5，你直接在token里面找byte pair，看看它们在不在merge rules里面。这样会快得多。

此外，我还弄了个LRU的小缓存~~(绝对不会告诉你这是因为之前弄成遍历一遍merge rules，导致这个函数太慢了)~~，储存token和编码结果，如果缓存命中就可以直接返回，不用再处理一遍。

指引上要求写的encode()函数和encode_iter函数，直接写就好了。它们适合用来处理小段形式的文本，约10M左右。

这一部分的最终任务，是将四个给出的数据集分别编码成为numpy uint16列表的形式。为高效处理这种文件形式的大文本编码，我专门写了一个`encode_files.py`。这个脚本专门负责大文件的并行化编码。操作如下：

1. 将文件预切分为若干个大小约为1M的块(切点在Special Tokens处)，这点和PreTokenize的操作很像。
2. 并行化编码所有块，保存成临时文件，避免过多内存占用。
3. 等所有块编码成临时文件后，合并所有临时文件。为了快速合并，合并的时候我们也采用并行化的设计，采用类似二叉树的合并方法。等剩余文件数较少或单个文件过大的时候，再采用串行合并。

整体速度：编码阶段3.5mins，合并阶段0.5mins。(owt_train.txt，最终编码结果owt_train_encoded.npy，6.5G)



## 实现语言模型的基本组件，并合成TransformerLM类

> 训练一个大语言模型需要几步？
>
> 第一步：从大语言模型类里声明一个对象
>
> 第二步：把收集到的数据处理好
>
> 第三步：Model.train(dataset)

这章的主要目的是构建一个后续我们可以点击即用的TransformerLM类。按照讲义上的介绍，基本架构如下左图所示。其核心组件是若干个Transformer块，一个Transformer块是由残差连接的方式组成的，包含一个多头注意力层和一个双线性层。

每个组件的具体实现，在讲义上已经介绍得很清楚了。我们这节主要从理论(或者直观)的角度来理解一下每个组件。

![image-20260210165720689](assets/image-20260210165720689.png)

### 基本架构


模型的输入是一段长至多为`max_seq_len`的token序列。训练时，模型会对每个i，仅依靠前`i`个token，预测第`i+1`个token的值。

首先，模型收到token序列`(...,seq_len)`后，经过Embedding层将每个Token编码成`d_model`维的向量。然后进入模型的核心，即多个堆叠的注意力层。我们的实现中，每个注意力层的输入和输出维度都是`(...,seq_len,d_model)`。模型的输出表示概率，经过softmax后计算loss。(即cross entropy loss)。

接下来我们来拆解一下一个Transformer Block中的具体组成：

### 注意力块的具体实现

> We offer no explanation as to why these architectures seem to work; we attribute their success, as all else, to divine benevolence.  ——Shazeer
#### RMS PreNorm without transition

一个注意力块包括一个多头注意力层和一个双线性层。层之间通过残差连接，且使用Pre-Norm的方式。

> A variety of work has found that moving layer normalization from the output of each sub-layer to the input of each sub-layerimproves Transformer training stability.

RMSNorm(Root mean square)正则化层的设计引入了权重参数$g_i$。公式如下：
$$
Norm(a_i)=\frac{a_i}{QM(a)+\epsilon}g_i
$$
其中QM是平方平均值(Quadratic Mean)，$g_i$是可学习的参数。

有点奇怪的是，与传统的Normalize相比，这里我们并没有用到平移来把均值归零。

#### 激活函数SiLU与SwiGLU

激活函数使用的是$SiLU:f(x)=\frac{x}{1+e^{-x}}=x\cdot\sigma(x)$。它和Relu的函数走势很相似，但相比Relu，SiLU在衔接处是平滑的。

![image-20260210175132516](assets/image-20260210175132516.png)

传统的双线性层采用`Linear-activate-Linear`的方式。通常来说，隐藏层的大小会设计成输入层的四倍。而教程推荐采用的办法是GLU(Gated Linear Units)的办法，这种方法也是采用两个线性变换，但是是使用逐项相乘（$\odot$）的形式。
$$
GLU(x,W_1,W_2)=f(W_1x)\odot (W_2x)
$$
GLU的想法是一个“门控机制”。上面的f是激活函数。最开始提出GLU的时候人们采用的是sigmoid函数，它输出0-1的一个数，逐项乘后面的线性层结果，直观上就代表在什么程度上保存对应的元素值。这相当于能够灵活地调整不同部分的权重。后面人们将其它激活函数(如Relu,SiLU等)替换sigmoid后，发现依然能够有良好的效果，直观上理解，它们也有类似"门控"的功能，能够抑制那些负数的部分。

最终我们的前向线性层FFN采用SwiGLU设计，公式如下：

![image-20260210203605968](assets/image-20260210203605968.png)

> Shazeer [2020] first proposed combining the SiLU/Swish activation with GLUs and conducted experiments showing that SwiGLU outperforms baselines like ReLU and SiLU (without gating) on language modeling tasks.  

#### RoPE传递相对位置信息

接下来是RoPE。这也是一个比较精妙的设计。我们在注意力机制中如果不加任何预处理，直接对所有的token统一进行注意力机制的处理，就会导致相同的token在不同的位置上的地位是等价的。换句话说，我们丢失了位置信息。

解决方式，当然是我们需要想办法将位置编码到向量中。一个朴素的想法是将位置(整数)作为一个额外的维度放在编码中。但是这带来的问题是这个编码维度会很大程度上破坏层内向量的数值分布(神经网络内，典型值一般差不多是标准正态分布)，同时也难以传递有效的信息。
RoPE的处理是将每个q和k向量(假设是`d_model`维)都看成在复数域上的`d_model/2`维的向量(每相邻两个元素组成一个复数的实部和虚部)，然后对每个复数维，设定一个角度$\theta_k(k=1,2\dots d_{model}/2)$，然后对序列中每个的token，让这个维度的复数旋转角度$i\theta_k(i=0,1,2\dots seq\_len-1)$。我们将RoPE应用在Q和K矩阵上，一个关键的特性是对于两个token i和j，则$q_i^T\cdot k_j$中收到旋转的角度值的影响只与$i-j$有关。这也就是说，在注意力机制中使用$Q^TK$的时候，实际上RoPE传递的是token的相对位置信息。这也是很符合我们语言直觉的一个结果。

一般来说，我们设置一个常数$\Theta$(常为10000)，并使$\theta_k=\Theta^{-2k/d}$。同时为了避免多次计算三角函数值拉慢速度，我们计算一次之后就会作为RoPE的类缓存，把三角函数值存起来。

#### 多头注意力机制

单头注意力的公式是
$$
Attention(Q,K,V)=softmax(\frac{Q^TK}{\sqrt{d_k}})V
$$
其中

![image-20260210213949993](assets/image-20260210213949993.png)

而多头注意力，在实现上就是把Q,K,V的$d_k,d_v$维度切成若干份，形成若干个头，分别去作注意力，最终拼合起来。这个好处是对于每个query，可以从不同的角度，分别给与到不同的keys以不同的权重分布。

整体的公式写成下面的形式：

![image-20260210214850650](assets/image-20260210214850650.png)

![image-20260210214926695](assets/image-20260210214926695.png)

其中$W_O$是最终Output前的线性变换，$Q_i,K_i,V_i$是Q,K,V矩阵的$d_k/d_v$维度按照头切块的结果，Concat就是将他们重新接连在一起。

此外，还记得我们说模型在训练时需要通过前i个token来预测第i+1个token。为了不让模型在训练时提前看到后面的序列内容，我们在注意力机制中会将对后面的Value的权重设置为0，具体而言，可以将$Q^TK$的结果中对应的部分设置成$-\inf$。

将上面的内容综合起来，就可以得到一个Transformer LM类。
