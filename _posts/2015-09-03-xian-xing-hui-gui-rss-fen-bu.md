---
layout: post
title: "线性回归RSS分布"
description: ""
category: ""
tags: []
---
{% include JB/setup %}
n为观察数    

p为特征数

线性模型:    

$$ {y} = X \beta + \epsilon $$    

那我们的残差：    

$$ 
\hat{\epsilon} = y - X \hat{\beta} 
               = (I - X (X'X)^{-1} X') y
               = Q y
               = Q (X \beta + \epsilon) = Q \epsilon
$$    

其中 $$ Q = I - X (X^TX)^{-1} X^T $$    

显然 $$ \textrm{tr}(Q) = n - p $$ (循环置换下迹的不变性)      

且 $$ Q^T=Q=Q^2 $$ 其中 $$ Q $$ 的特征值为 $$ 0 $$ 和 $$ 1 $$     

那么存在一个酉矩阵 $$ V $$ 使得：    

$$ V^TQV = \Delta = \textrm{diag}(\underbrace{1, \ldots, 1}_{n-p-1 \textrm{ times}}, \underbrace{0, \ldots, 0}_{p \textrm{ times}}) $$     

([矩阵可以被酉矩阵对角化当且仅当它们是noramal的](http://en.wikipedia.org/wiki/Diagonalizable_matrix))     

我们让 $$ K = V^T \hat{\epsilon} $$    

由于 $$ \hat{\epsilon} \sim N(0, \sigma^2 Q) $$     

于是我们有 $$ K \sim N(0, \sigma^2 \Delta) $$     

其中 $$ K_{n-p}=\ldots=K_n=0 $$ 

所以 $$ \frac{\|K\|^2}{\sigma^2} = \frac{\|K^{\star}\|^2}{\sigma^2} \sim \chi^2_{n-p-1} $$    

其中 $$ K^{\star} = (K_1, \ldots, K_{n-p-1})^T $$    

其中 $$ V $$ 是酉矩阵，我们同样可以得到：    

$$ \|\hat{\epsilon}\|^2 = \|K\|^2=\|K^{\star}\|^2 $$     

所以

$$ \frac{\textrm{RSS}}{\sigma^2} \sim \chi^2_{n-p-1} $$    





因为 $$ Q^2 - Q =0, $$ Q的[最小化多项式](http://en.wikipedia.org/wiki/Minimal_polynomial_%28linear_algebra%29)除以多项式 $$ z^2 - z $$ 所以 $$ Q $$ 的特征值只有 $$ 0 $$ 或 $$ 1 $$ 。    
由于 $$ \textrm{tr}(Q) = n - p - 1 $$ 且特征值的和为矩阵的迹,所以我们有 $$ 1 $$ 为 $$ n - p - 1 $$ 重特征值 $$ 0 $$ 为 $$ p + 1 $$ 重特征值     


![参考](http://stats.stackexchange.com/questions/20227/why-is-rss-distributed-chi-square-times-n-p)
