---
layout: post
title: "神经网络  Backpropagation Algorithm"
description: ""
category: ""
tags: []
---
{% include JB/setup %}
###损失函数    
假设有 $$ m $$ 个训练样本，每个包含一组输入信号 $$ x $$ 和一组输出信号 $$ y $$, 
$$ L $$ 表示神经网络的层数。    
$$ S_l $$ 表示每层的神经元个数， $$ S_L $$ 表示输出层神经元个数。    
$$ K $$ 表示分为多少类。    

所以我们的损失函数为：    

$$ 
J(\theta) = -\frac{1}{m}\left[\sum_{i=1}^{m}\sum_{k=1}^Ky_k^{(i)}log(h_\theta(x^{(i)})_k + (1 - y^{(i)})log(1 - (h_\theta(x^{(i)}))_k)\right]
$$

###Backpropagation Algorithm    
![Back ]({{ BASE_PATH }}/assets/images/machineLearning/bp.png)      
对于上图我们有：    

$$ a^{(1)} = x $$    

$$ z^{(2)} = \Theta^{(1)}a^{(1)} $$    

$$ a^{(2)} = g(z^{(2)}) $$    

$$ z^{(3)} = \Theta^{(2)}a^{(2)} $$    

$$ a^{(3)} = g(z^{(3)}) $$    

$$ z^{(4)} = \Theta^{(3)}a^{(3)} $$    

$$ a^{(4)} = h_\Theta(x) = g(z^{(4)}) $$    

我们的总误差为：    

$$ E = \frac{1}{2}\sum_i(y_i - a_i)^2 $$    

对于最后一层，我们可以直接算出网络产生的输出 $$ a^{(L)} $$ 与实际值之间的差距。然后我们通过计算各层节点残差的加权平均值计算隐藏层的残差。    

我们把每层的差值标记为 $$ \delta_i^{(l)} $$ 那么我们就有：    

在最后一层中

$$
\delta_{i}^{(l)} = \frac{\partial E}{\partial z_i^{(l)}}=\frac{\partial \frac{1}{2}(y-a_i^{(l)})^2}{\partial z_i^{(l)}}=\frac{\partial [\frac{1}{2}(y-g(z_i^{(l)}))^2]}{\partial z_i^{(l)}}= (a_i^{(l)}-y)\cdot g{’}(z_i^{(l)})
$$

对于前面的每一层，都有

$$
\delta_{i}^{(l)} = \frac{\partial E}{\partial z_i^{(l)}}=\sum_{j}^{N^{(l+1)}} \frac{\partial E}{\partial z_j^{(l+1)}} \cdot \frac{\partial z_j^{(l+1)}}{\partial z_i^{(l)}}\\ = \sum_{j}^{N^{(l+1)}} \delta_{j}^{(l+1)}\cdot \frac{\partial [\sum_{k}^{N^{l}}\Theta_{jk}\cdot g(z_k^{(l)})] }{\partial z_i^{(l)}},i\in k\\ =\sum_{j}^{N^{(l+1)}} (\delta_{j}^{(l+1)}\cdot \Theta_{ji}) \cdot g{'}(z_i)
$$

由此得到第$$ l $$ 层第 $$ i $$ 个节点的残差计算方法：

$$
\delta_{i}^{(l)} =\sum_{j}^{N^{(l+1)}} (\delta_{j}^{(l+1)}\cdot \Theta_{ji}) \cdot g{'}(z_i)
$$

由于我们的真实目的是计算 $$ \frac{\partial E(\Theta)}{\partial \Theta_{ji}} $$ 且有：    

$$
\frac{\partial E}{\partial \Theta_{ji}^{l}} = \frac{\partial E}{\partial z_i^{(l+1)}}\cdot \frac {\partial z_i^{(l+1)}}{\partial \Theta_{ji}^{l}} =\delta_i^{(l+1)}\cdot a_j^{(l)}
$$

所以我们可以得到神经网络中权重的更新方程：    

$$
\Theta_{ji}^{l} = \Theta_{ji}^{l}-\alpha\cdot \delta_i^{(l+1)}\cdot a_j^{(l)}
$$
