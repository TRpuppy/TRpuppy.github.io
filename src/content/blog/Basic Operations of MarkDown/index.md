---
title: 'Basic Operations Of MarkDown' # (Required, max 60)
description: '准备开始使用MarkDown来写博客。这里就先发一篇记录一下MarkDown的学习心得。' # (Required, 10 to 160)
publishDate: '2025-11-12 00:17:00' # (Required, Date)
tags:
  - Markdown
heroImage: { src: './adf1d1ec5812ae93214987d03c70ea4.jpg', alt: '强！', color: '#B4C6DA' }
draft: false # (set true will only show in development)
language: 'Chinese' # (String as you like)
comment: true # (set false will disable comment, even if you've enabled it in site-config)
---

# MarkDown编辑器——Typora

我使用[**Typora**](https://typora.io/)来进行MarkDown的编辑。它是收费的，因为没搞定网络上的破解版，于是迫不得已支持正版:joy:



# MarkDown基本操作

## 标题

使用＃键可以创建标题格式的文字。例如创建一个一级标题，可以输入一个#，一个空格，再加上后面标题的文字。编辑器会自动将文字编译成标题。

```markdown
# 这是一个一级标题。
## 这是二级标题。
这就是正常的文本。
```

**如果不打空格，则会被识别为正文。**
MarkDown有六级标题。输入时按照#的个数来区分。
此外，还有额外的快捷键方法可以调整标题。

- ctrl+数字0-6可以将选中文本调整成对应级别的标题。其中0级标题即为正文，1级标题是最大的。
- ctrl+加减号可以调整选中文本的标题级别。

## 段落

MarkDown中的换行分为两种。一种是大换行，直接输入enter就是大换行。

例如这一行与上一行就是大换行。
shift+enter是小换行。这一行和上一行之间就是小换行。
大换行与小换行的区别是大换行在md代码中是换两行。
我不喜欢换两行。觉得行距太大了。所以我一般都会输入shift+enter换行。
分割线的输入是三个减号。效果就是下面这个样子。

---

## 文字

### 1.基本文字效果

基本的文字效果，在md里面都有代码对应。

```markdown
**这是粗体效果**
~~删除线~~
<u>下划线</u>
*斜体*
***粗体加斜体***
==高亮==
```

**这是粗体效果**
~~删除线~~
<u>下划线</u>
*斜体*
***粗体加斜体***
==高亮==

此外还有快捷键操作。

- Ctrl+B:选中的文本加粗。
- Shift+Alt+5:选中的文本加删除效果。
- Ctrl+U:下划线。
- Ctrl+L:斜体效果。

此外，可以使用反斜杠来取消转义效果。

```markdown
1\*2\*3\*4
```

1\*2\*3\*4

### 2.上下标效果

其实不是很喜欢使用MarkDown自带的上下标功能~小声逼逼~
上下标的表示方式是^和~。

```markdown
^上标^
~下标~
```

为什么不喜欢它的上下标呢，因为本来Latex就有上下标，且md显示的上下标还会有显示问题。例如x~1~^2^,它并没有按照期望的显示。而用Latex——$x_1^2$就会漂亮很多。

## 列表

### 1.无序列表

输入*或+或-再加空格，均可进入无序列表状态。
直接enter换行就可以开始下一项，按两次enter可以退出列表状态。

+ 列表第一项。
+ 第二项。

