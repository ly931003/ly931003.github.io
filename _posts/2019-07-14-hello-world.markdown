---
layout:     post
title:      "BP人工神经网络的数学原理推导"
subtitle:   "果然还是数学推导比较有意思，反复推几遍怎么都忘不了了"
date:       2019-07-14 16:37:28
author:     "Yubari Melon"
header-img: "img/post-bg-2015.jpg"
tags:
    - 人工智能
    - 算法推导
---

> “Yeah It's on. ”

$$ h_j=\sum_iw_{ij} x_i+b_j
\\H_j = \sigma(h_j)
\\这里\ \sigma(x) = \frac{1}{1+e^{-x}}
\\有\  \sigma(x)^{'} = \frac{e^{-x}}{(1+e^{-x})^2}=\sigma(x)[1-\sigma(x)]
\\o_k=\sum_jv_{jk} H_j+c_k
\\O_k = \sigma(o_k)
\\E(O) = \frac{1}{2} \sum_k (Y_k-O_k)^2
\\\frac{\partial E}{\partial v_{jk}} = \frac{\partial E}{\partial O_k} \frac{\partial O_k}{\partial o_k} \frac{\partial o_k}{\partial v_{jk}}
\\\frac{\partial E}{\partial c_k} = \frac{\partial E}{\partial O_k} \frac{\partial O_k}{\partial o_k} \frac{\partial o_k}{\partial c_k}
\\ \because \frac{\partial E}{\partial O_k} = O_k- Y_k\quad \frac{\partial O_k}{\partial o_k} =O_k(1-O_k) \quad \frac{\partial o_k}{\partial v_{jk}}=H_j \quad \frac{\partial o_k}{\partial c_k}=1
\\\therefore \frac{\partial E}{\partial v_{jk}} = (O_k- Y_k)[O_k(1-O_k)]H_j
\\ \frac{\partial E}{\partial c_k}=(O_k- Y_k)[O_k(1-O_k)]
$$

$同理有$

$$
\frac{\partial E}{\partial w_{ij}} = \sum_k \frac{\partial E}{\partial O_k} \frac{\partial O_k}{\partial o_k} \frac{\partial o_k}{\partial H_j} \frac{\partial H_j}{\partial h_j} \frac{\partial h_j}{\partial w_{ij}} = \{\sum_k (O_k- Y_k)[O_k(1-O_k)]v_{jk}\}[H_j(1-H_j)]x_i
\\ \quad \frac{\partial E}{\partial b_j} = \sum_k \frac{\partial E}{\partial O_k} \frac{\partial O_k}{\partial o_k} \frac{\partial o_k}{\partial H_j} \frac{\partial H_j}{\partial h_j} \frac{\partial h_j}{\partial b_j} = \{\sum_k (O_k- Y_k)[O_k(1-O_k)]v_{jk}\}[H_j(1-H_j)]
$$

$不妨令$

$$
\varepsilon_k =  (O_k- Y_k)[O_k(1-O_k)]
\\\frac{\partial E}{\partial v_{jk}} = \varepsilon_kH_j \quad \frac{\partial E}{\partial c_k}=\varepsilon_k
\\\frac{\partial E}{\partial w_{ij}} = (\sum_k \varepsilon_kv_{jk})[H_j(1-H_j)]x_i \quad \frac{\partial E}{\partial b_j} = (\sum_k \varepsilon_kv_{jk})[H_j(1-H_j)]
$$

$再令$

$$
\delta_j = (\sum_k \varepsilon_kv_{jk})[H_j(1-H_j)]
\\\frac{\partial E}{\partial w_{ij}} = \delta_jx_i \quad \frac{\partial E}{\partial b_j} = \delta_j
$$

$根据梯度下降法可知$

$$
\pmb{v_{jk}}  = v_{jk}-\gamma \varepsilon_kH_j \quad \pmb{ c_k} = c_k-\gamma \varepsilon_k
\\ \pmb{w_{ij}} = w_{ij}-\gamma \delta_jx_i \quad \pmb{b_j} =b_j-\gamma \delta_j
$$

$其中\gamma被称为学习率，可以通过设置\gamma来调节梯度下降的速率。$