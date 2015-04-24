---
layout: post
title: "线性回归"
description: ""
category: ""
tags: []
---
{% include JB/setup %}
##线性回归模型    
机器学习算法的主要目的是找到能够对数据做出最优解释的模型,一般有以下步骤：    
1. 假设函数    
2. 建立假设函数的评估标准（一般使用损失函数作为评估标准）    
3. 根据评估标准递推出目标函数(也就是根据评估标准找到目标函数的最优解)      

具体到线性回归就是：    
一个多变量线性回归模型表示为以下形式：    

$$
\begin{align*}
Y_i = \beta_0 + \beta_1 X_{i1} + \beta_2 X_{i2} + \ldots + \beta_p X_{ip} + \varepsilon_i, \qquad i = 1, \ldots, n 
\end{align*}
$$

使用矩阵表示为：    

$$
\begin{align*}
Y = X \beta + \varepsilon \,
\end{align*}
$$

其中 $$ Y $$ 是一个包括了观测值 $$  Y_1, \ldots, Y_n $$ 的列向量, $$  \varepsilon $$ 包括了未观测的随机变量 $$  \varepsilon_1, \ldots, \varepsilon_n $$ 以及回归量的观测矩阵 $$ X $$

$$
\begin{align*}
X = \begin{pmatrix} 1 & x_{11} & \cdots & x_{1p} \\ 1 & x_{21} & \cdots & x_{2p}\\ \vdots & \vdots & \ddots & \vdots \\ 1 & x_{n1} & \cdots & x_{np} \end{pmatrix}
\end{align*}
$$

回归分析的最初目的是估计模型的参数以便对达到对数据的最佳拟合。    
在决定一个最佳拟合的不同标准中，最小二乘法是非常优越的。    
这种估计可表示为：    

$$  \hat\beta = (X^T X)^{-1}X^T y \, $$

它通过最小误差的平方和寻找数据的最佳函数匹配。

###优点    
1. 不需要选择 $$ \alpha $$     
2. 不需要迭代    


###缺点    
由于最小二乘法的复杂度为 $$ O\left(n^3\right) $$ 当我们有很多变量的这个方法就不太适用。这时我们就需要用到
[梯度下降](/2015/04/17/ti-du-xia-jiang/)。
