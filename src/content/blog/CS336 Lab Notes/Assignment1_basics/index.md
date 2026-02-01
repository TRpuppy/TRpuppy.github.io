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

这个函数的具体实现：别傻傻地去遍历一遍所有merge rules(约32000个)了~~(没错说得就是我自己)~~，一个token大小约为5，你直接在token里面找byte pair，看看它们在不在merge rules里面。这样会快得多。此外，我还弄了个LRU的小缓存~~(绝对不会告诉你这是因为之前弄成遍历一遍merge rules，导致这个函数太慢了)~~，储存token和编码结果，如果缓存命中就可以直接返回，不用再处理一遍。

指引上要求写的encode()函数和encode_iter函数，直接写就好了。它们适合用来处理小段形式的文本，约10M左右。

这一部分的最终任务，是将四个给出的数据集分别编码成为numpy uint16列表的形式。为高效处理这种文件形式的大文本编码，我专门写了一个`encode_files.py`。这个脚本专门负责大文件的并行化编码。操作如下：

1. 将文件预切分为若干个大小约为1M的块(切点在Special Tokens处)，这点和PreTokenize的操作很像。
2. 并行化编码所有块，保存成临时文件，避免过多内存占用。
3. 等所有块编码成临时文件后，合并所有临时文件。为了快速合并，合并的时候我们也采用并行化的设计，采用类似二叉树的合并方法。等剩余文件数较少或单个文件过大的时候，再采用并行合并。

整体速度：编码阶段3.5mins，合并阶段0.5mins。(owt_train.txt，最终编码结果owt_train_encoded.npy，6.5G)