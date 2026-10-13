---
title: 电路分析基础学习知识点汇总
date: 2026-10-13 20:01:00
tags: [Concepts, Knowledge]
categories:
  - 电路分析
---

本文主要汇总一些《电路基础》这本书描述的部分知识点，不间断更新。

<!-- more -->
## 正弦量与相量

### 正弦信号
正弦信号是指具有正弦或者余弦函数形式的信号。

### 正弦电压
正弦电压可以表示为：  
  - $v(t) = V_m \sin(\omega t)$

其中：  
  - $V_m$ 为正弦电压的幅度或者振幅（amplitude）
  - $\omega$ 为角频率，单位为 rad/s（angular frequency）
  - $\omega t$ 为正弦电压的幅角（argument）

正弦电压每隔 $T$ 秒就会重复一遍，所以称 $T$ 为正弦电压的周期（period），且 $\omega T = 2\pi$，即 $T = \frac{2\pi}{\omega}$。  
由于 T 为周期，则又有 $v(t + nT) = v(t)$。
周期的倒数即为频率，故有$f = \frac{1}{T}$，所以 $\omega = 2\pi f$。频率的单位为赫兹（Hz）,表示每秒循环个数。

### 正弦电压的一般表达式
正弦电压的一般表达式：  
  - $v(t) = V_m \sin(\omega t + \phi)$
  
### 相量
相量是一个表示正弦信号的幅度和相位的复数。相量提供了一种分析正弦电源激励的线性电路的简单办法，否则这类电路的求解会很困难。
#### 欧拉恒等式
  - $e^{\pm j\phi} = \cos\phi \pm j\sin\phi$

由上可知，$\cos\phi$ 和 $\sin\phi$ 可分别看作 $e^{j\phi}$ 的实部和虚部，即：
  - $\cos\phi = Re(e^{j\phi})$
  - $\sin\phi = Im(e^{j\phi})$

所以，已知正弦信号 $v(t) = V_m \cos(\omega t + \phi)$，由上式可得  
  - $v(t) = Re(V_m e^{j(\omega t + \phi)})$
  - $v(t) = Re(V_m e^{j\phi} e^{j\omega t})$
  - $v(t) = Re(V e^{j\omega t})$，其中 $V = V_m e^{j\phi} = V_m \angle \phi$

### 电路元件的相量关系

#### 电阻
电阻在相量域中的电压电流关系服从欧姆定律。假设流过电阻R的电流为 $i = I_m \cos(\omega t + \phi)$  
由欧姆定律可知，其两段的电压为 $v = iR = RI_m\cos(\omega t + \phi)$  
在相量域中表示为 $V = RI_m \angle \phi$


