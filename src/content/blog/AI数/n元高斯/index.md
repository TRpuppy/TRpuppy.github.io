---
title: 'Prove some properties of n-variable Gaussian distribution' # (Required, max 60)
description: '也许我一生涓滴意念，侥幸汇成河——李宗盛《山丘》' # (Required, 10 to 160)
publishDate: '2025-11-13 00:16:00' # (Required, Date)
tags:
  - Markdown
heroImage: { src: './1b52df7d8e194a5b707f2b22e2cb5d2.png', alt: '比起阳光，我更需要阳菜！', color: '#B4C6DA' }
draft: false # (set true will only show in development)
language: 'Chinese' # (String as you like)
comment: true # (set false will disable comment, even if you've enabled it in site-config)
---

# 关于N元高斯分布的一些性质证明

## N元高斯分布的定义

称n维度随机向量$\eta=(X_1,X_2,...,X_n)$服从n元正态分布，若$\eta$有以下的联合分布:
$$
\Large p(x)=\frac{1}{(2\pi)^{\frac{n}{2}}|\Sigma|^{\frac{1}{2}}}exp(-\frac{1}{2}(x-\mu)\Sigma^{-1}(x-\mu)^T)
$$
其中$x,\mu$为n维行向量，$\Sigma$为n*n的正定矩阵。

下面来证明该定义的合理性与其性质。

## 定义的合理性

所谓定义的合理性，就是证明概率的归一化性质。即需要证明:
$$
\Large\int_{x_1=-\infty}^{+\infty}\int_{x_2=-\infty}^{+\infty}...\int_{x_n=-\infty}^{+\infty}p(x_1,x_2,...x_n)dx_1dx_2...dx_n=1
$$
证明：我们来作归纳证明。奠基是显然的。假设对于n-1维的正态分布，结论成立，则对于n元正态分布：
$$
\begin{align}
& \int_{-\infty}^{+\infty}...\int_{-\infty}^{+\infty}p(x_1,x_2,...x_n)dx_1...dx_n\\
=&\frac{1}{(2\pi)^{\frac{n}{2}}|\Sigma|^{\frac{1}{2}}}\int_{-\infty}^{+\infty}...\int_{-\infty}^{+\infty}exp(-\frac{1}{2}(x-\mu)\Sigma^{-1}(x-\mu)^T)dx_1...dx_n\\
\end{align}
$$
设$\Sigma^{-1}=\begin{pmatrix}
\Sigma_{n-1}^{-1} & \alpha \\
\alpha^T & c
\end{pmatrix},x=\begin{pmatrix}x^{\prime} & x_n\end{pmatrix}$,并不妨令$\mu=0$

于是
$$
原式=\frac{1}{(2\pi)^{\frac{n}{2}}|\Sigma|^{\frac{1}{2}}}\int_{-\infty}^{+\infty}...\int_{-\infty}^{+\infty}exp(-\frac{1}{2}x^{\prime}\Sigma^{\prime -1}x^{\prime T})[\int_{-\infty}^{+\infty}exp(-\frac{1}{2}(cx_n^2+2x^{\prime}\alpha ))dx_n] dx_1...dx_{n-1}\\
$$
其中
$$
\begin{align}
&\int_{-\infty}^{+\infty}exp(-\frac{1}{2}(cx_n^2+2x^{\prime}\alpha ))dx_n\\
=&\int_{-\infty}^{+\infty}exp(-\frac{1}{2}c(x_n^2+2\frac{1}{c}x^{\prime}\alpha +(\frac{1}{c}x^{\prime}\alpha )^2))\times exp(\frac{1}{2c}(x^{\prime}\alpha )^2)dx_n\\
=&\int_{-\infty}^{+\infty}exp(-\frac{1}{2}c(x_n+\frac{1}{c}x^{\prime}\alpha )^2)\times exp(\frac{1}{2c}x^{\prime}\alpha\alpha^Tx^{\prime T}))dx_n\\
=&\sqrt{\frac{2\pi}{c}}\times exp(\frac{1}{2c}x^{\prime}\alpha\alpha^Tx^{\prime T}))\\
\end{align}
$$
故
$$
\begin{align}原式
&=\frac{\sqrt{\frac{2\pi}{c}}}{(2\pi)^{\frac{n}{2}}|\Sigma|^{\frac{1}{2}}}\int_{-\infty}^{+\infty}...\int_{-\infty}^{+\infty}exp(-\frac{1}{2}x^{\prime}\Sigma^{\prime -1}x^{\prime T}+\frac{1}{2c}x^{\prime}\alpha\alpha^Tx^{\prime T})dx_1...dx_{n-1}\\
&=\frac{\sqrt{\frac{2\pi}{c}}}{(2\pi)^{\frac{n}{2}}|\Sigma|^{\frac{1}{2}}}\int_{-\infty}^{+\infty}...\int_{-\infty}^{+\infty}exp(-\frac{1}{2}x^{\prime}(\Sigma^{\prime -1}+\frac{\alpha\alpha^T}{c})x^{\prime T})dx_1...dx_{n-1}\\
\end{align}
$$
由归纳假设
$$
\begin{align}原式
&= \frac{\sqrt{\frac{2\pi}{c}}}{(2\pi)^{\frac{n}{2}}|\Sigma|^{\frac{1}{2}}}(2\pi)^{\frac{n-1}{2}}|\Sigma^{\prime -1}+\frac{\alpha\alpha^T}{c}|^{-\frac{1}{2}}\\
&= \sqrt{\frac{|\Sigma^{-1}|}{c|\Sigma^{\prime -1}+\alpha\alpha^T|}}\\
\end{align}
$$
由基本初等行变换可知$|\Sigma^{-1}|=c|\Sigma^{\prime -1}+\alpha\alpha^T|$。于是$原式=1$。$\square$

## 期望值

我们证明$E(x)=\mu$。

相似地，我们使用归纳法证明，采用与上面相同的记号，并依然不妨设$\mu=0$.于是就只需要证明$E(x)=0$

奠基是显然的。假设对于n-1维的正态分布，结论成立，则对于n元正态分布，不失一般性地，我们去求$E(x_n)$。
$$
\begin{align}E(x_n)&= \int_{-\infty}^{+\infty}...\int_{-infty}^{+\infty}p(x_1,x_2,...x_n)x_ndx_1...dx_n\\
&=\frac{1}{(2\pi)^{\frac{n}{2}}|\Sigma|^{\frac{1}{2}}}\int_{-\infty}^{+\infty}...\int_{-\infty}^{+\infty}exp(-\frac{1}{2}x^{\prime}\Sigma^{\prime -1}x^{\prime T})[\int_{-\infty}^{+\infty}x_nexp(-\frac{1}{2}(cx_n^2+2x^{\prime}\alpha ))dx_n] dx_1...dx_{n-1}\\
\end{align}
$$
其中，
$$
\begin{align}
&\int_{-\infty}^{+\infty}x_nexp(-\frac{1}{2}(cx_n^2+2x^{\prime}\alpha ))dx_n\\
=&\int_{-\infty}^{+\infty}x_nexp(-\frac{1}{2}c(x_n+\frac{1}{c}x^{\prime}\alpha )^2)\times exp(\frac{1}{2c}x^{\prime}\alpha\alpha^Tx^{\prime T}))dx_n\\
=&-\frac{x^{\prime}\alpha}{c}\times exp(\frac{1}{2c}x^{\prime}\alpha\alpha^Tx^{\prime T}))\\
\end{align}
$$
故由归纳假设，有
$$
\begin{align}原式
&=\frac{1}{(2\pi)^{\frac{n}{2}}|\Sigma|^{\frac{1}{2}}}\int_{-\infty}^{+\infty}...\int_{-\infty}^{+\infty}-\frac{x^{\prime}\alpha}{c}exp(-\frac{1}{2}x^{\prime}\Sigma^{\prime -1}x^{\prime T}+\frac{1}{2c}x^{\prime}\alpha\alpha^Tx^{\prime T})dx_1...dx_{n-1}\\
&=0\quad \quad \quad \square\\

\end{align}
$$
