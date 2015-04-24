---
layout: post
title: "梯度下降"
description: ""
category: ""
tags: []
---
{% include JB/setup %}
梯度下降法是一个最优化算法，通常也称为最速下降法。

###描述
梯度下降法，基于这样的观察：    

如果实值函数 $$ F(\mathbf{x}) $$ 在点 $$ a $$ 处可微且有定义，那么函数 $$ F(\mathbf{x}) $$ 在 $$ a $$ 点沿着梯度相反的方向 $$ -\nabla F(\mathbf{a}) $$ 下降最快。    

因而，如果    

$$ 
\begin{align*}
\mathbf{b}=\mathbf{a}-\gamma\nabla F(\mathbf{a}) 
\end{align*}
$$     


当 $$ \gamma>0 $$ 为一个足够小的数值时$$ F(\mathbf{a})\geq F(\mathbf{b}) $$ 成立。     


考虑到这一点，我们可以从函数 $$ F $$ 的局部极小值的初始估计 $$ \mathbf{x}_0 $$ 出发，并考虑如下序列 $$ \mathbf{x}_0, \mathbf{x}_1, \mathbf{x}_2, \dots $$ 使得：    


$$ 
\begin{align*}
\mathbf{x}_{n+1}=\mathbf{x}_n-\gamma_n \nabla F(\mathbf{x}_n),\ n \ge 0. 
\end{align*}
$$    

因此可得到    

$$ 
\begin{align*}
F(\mathbf{x}_0)\ge F(\mathbf{x}_1)\ge F(\mathbf{x}_2)\ge \cdots, 
\end{align*}
$$     

如果顺利的话序列 $$ (\mathbf{x}_n) $$ 收敛到期望的极值。(每次迭代步长 $$ \gamma $$ 是可以改变的)    

###算法实现    
对于输入样本集 $$ \mathbf{x}^{(i)}, \mathbf{y}^{(i)}; i=1,\dots,m, $$ 其中 $$ x^i=\left[ x_0^{(i)}, \dots, x_n^{(i)} \right]^T $$ 上标为样本序号，共 $$ m $$ 个样本，下标为每个样本的特征数，共 $$ n $$ 个特征。    

我们的假设函数为：    

$$ h(x) = \sum_{i=0}^n \theta_iX_i = \theta^Tx $$    

那我们的损失函数为：    

$$ J(\theta) = \frac{1}{2}\sum_{i=1}^m(h_\theta(x^{(i)}) - y^{(i)} )^2 $$    

我们的目标就是找到参数向量 $$ \theta = [\theta_0, \dots, \theta_n]^T $$ 的值，使得 $$ J(\theta) $$ 最小。    

我们的步骤为：    

1. 选取参数向量的初始值     
2. 对于每个参数 $$ \theta_j (j \in [0,n]) $$的值做如下更新：    

$$ \theta_j := \theta_j - \alpha\frac{\partial}{\partial \theta_j}J(\theta) $$     

将 $$ J(\theta) $$ 带入上式，则有    

$$ \theta_j := \theta_j + \alpha\sum_{i=1}^m (y^{(i)} - h_{\theta}(x^{(i)}))x_j^{(i)}  (for\ every\ j) $$ 

由于该算法每次更新 $$ \theta $$ 中的元素，需要处理整个输入样本集，所有该算法也叫批量梯度下降(batch gradient descent)。当参数向量 $$ \theta $$ 中所有元素都更新之后代入 $$ J(\theta) $$ ，通过两次迭代值来判断其是否收敛以终止迭代。


