---
title: 电路分析基础学习知识点汇总
date: 2026-10-13 20:01:00
tags: [Concepts, Knowledge]
categories:
  - 电路分析
---

本文主要汇总一些电路分析相关的部分知识点，不间断更新。

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

#### 电感
对于电感而言，假设流过电感的电流为 $i = I_m \cos(\omega t + \phi)$  
则电感两端的电压为 $v = L \frac{di}{dt} = -\omega L I_m \sin(\omega t + \phi) = \omega L I_m \cos(\omega t + \phi + \frac{\pi}{2})$  
在相量域中表示则有 $V = \omega L I_m \angle \phi + \frac{\pi}{2}$  

令 $I = I_m \angle \phi$ 且由欧拉恒等式得 $e^{j \frac{\pi}{2}} = j$  
故有 $V = j \omega L I$

#### 电容
对于电容 C 而言, 假设电容两段的电压为 $v = V_m \cos(\omega t + \phi)$  
则流过电容的电流为 $i = C \frac{dv}{dt} = \omega C V_m \cos(\omega t + \phi + \frac{\pi}{2})$  
在向量域中表示则有 $I = \omega C V_m \angle \phi + \frac{\pi}{2}$  

令 $V = V_m \angle \phi$ 且由欧拉恒等式得 $e^{j \frac{\pi}{2}} = j$  
固有 $I = j \omega C V$  
则 $V = \frac{I}{j \omega C}$

#### 无源元件的相量关系
| 电路元件 | 时域  | 相量域  |
|  ----   | ----  | ----  |
| 电阻	  | $v = iR$ | $V = IR$ |
| 电感	  | $v = L \frac{di}{dt}$ | $V = j \omega L I$ |
| 电容	  | $i = C \frac{dv}{dt}$ | $I = j \omega C V$  |

### 阻抗与导纳

#### 阻抗
电路的阻抗是指相量电压 $V$ 和相量电流 $I$ 之比，单位欧姆($\Omega$)。

#### 导纳
导纳为阻抗的倒数，单位西门子($S$)。

#### 无源元件的阻抗与导纳
| 电路元件 | 阻抗  | 导纳  |
|  ----   | ----  | ----  |
| 电阻	  | $Z = R$ | $Y = \frac{1}{R}$ |
| 电感	  | $Z = j \omega L$ | $Y = \frac{1}{j \omega L}$ |
| 电容	  | $Z = \frac{1}{j \omega C}$ | $Y = j \omega C$  |

#### 阻抗和导纳的直角坐标形式表达
阻抗作为一个复数，可以用直角坐标形式表示为： $Z = R + j X$  
其中，R 为电阻，X 为电抗。  

同理，导纳也可以用直角坐标形式表示： $Y = G + j B$  
其中，G 为电导，B 为电纳。

### 阻抗合并
N个串联阻抗的等效阻抗，等于各个阻抗之和：$Z_{eq} = Z_1 + Z_2 + Z_3 + Z_4 + \cdots + Z_N$  

N个并联阻抗的等效阻抗，表示为：$\frac{1}{Z_{eq}} = \frac{1}{Z_1} + \frac{1}{Z_2} + \frac{1}{Z_3} + \cdots + \frac{1}{Z_N}$  
如果用导纳来表示，则有 $Y_{eq} = Y_1 + Y_2 + Y_3 + Y_4 + \cdots + Y_N$

## 正弦稳态分析

### 分析方法
1. 将电路转换到相量域或频域；
2. 利用相应的电路分析方法（节点分析法、网孔分析法、叠加定理等）求解电路；
3. 将所求得的相量转换到时域；
4. 由于 KCL 和 KVL 在交流电路中也是有效的，求解时可以灵活运用基尔霍夫电流和电压定律。

## 交流功率分析

### 瞬时功率和平均功率

#### 瞬时功率的概念
瞬时功率是指任一瞬间的功率，单位为瓦特。瞬时功率是元件吸收能量的速率。
元件吸收的瞬时功率$p(t)$等于改元件两端的瞬时电压$v(t)$与流经该元件的瞬时电流i(t)的乘积。假设采用无源符号的国际惯例，则有：$p(t) = v(t)i(t)$

#### 瞬时功率的一般情况
设电路的终端电压和电流分别为：  
$v(t) = V_m \cos(\omega t + \theta_v)$  
$i(t) = I_m \cos(\omega t + \theta_i)$  

由 $p(t) = v(t)i(t)$ 和三角恒等式推得 $p(t) = \frac{1}{2} V_m I_m \cos(\theta_v - \theta_i) + \frac{1}{2} V_m I_m \cos(2 \omega t + \theta_v + \theta_i)$  
上式表明，瞬时功率包含与时间无关的常量部分和与时间有关的正弦函数部分，正弦函数的角频率为$2\omega$。

瞬时功率随时间变化，一般难以测量。

#### 平均功率的概念
平均功率是指一个周期内瞬时功率的平均值，单位瓦特。平均功率可以表示为 $P = \frac{1}{T} \int_{0}^{T} p(t) dt = \frac{1}{2} V_m I_m \cos(\theta_v - \theta_i)$  
由上可知，平均功率与时间无关。

电阻性负载(R)在任何时刻均吸收功率，而电抗负载(L或C)吸收的平均功率为零。

### 最大平均功率传输
为了实现最大平均功率传输，负载阻抗必须满足$X_L = -X_{Th}$ 且 $R_L = -R_{Th}$，即：$Z_L = R_L + j X_L = R_{Th} - j X_{Th} = Z_{Th}^{*}$，故作如下描述：  
对于最大平均功率传输而言，负载阻抗 Z_L 必须等于戴维南阻抗 Z_{Th} 的共轭复数。当$R_L = R_{Th}且X_L = -X_{Th}$时，得到最大平均功率：$P_{max} = \frac{|V_{Th}| ^ 2}{8 R_{Th}}$  
在负载为纯实数的情况下，令 $X_L = 0$，得到最大功率传输条件为：$R_L = \sqrt{R_{Th}^2 + X_{Th} ^ 2} = |Z_{Th}|$，负载阻抗（电阻）等于戴维南阻抗的模。

### 有效值
周期性信号的有效值就是它的方均根值（rms）。 如下所示：    
  - $I_{eff} = \sqrt{\frac{1}{T} \int_{0} ^ {T} i^2 dt}$ &emsp; 周期性电流的有效值
  - $V_{eff} = \sqrt{\frac{1}{T} \int_{0} ^ {T} v^2 dt}$ &emsp; 交流电压的有效值
  - $X_{rms} = \sqrt{\frac{1}{T} \int_{0} ^ {T} x^2 dt}$ &emsp; 周期性信号的方均根

对于正弦信号 $i(t) = I_m \cos(\omega t)$ 和 $v(t) = V_m \cos(\omega t)$，有：  
  - $I_{rms} = \frac{I_m}{\sqrt{2}}$
  - $V_{rms} = \frac{V_m}{\sqrt{2}}$

### 视在功率
视在功率（单位为$V \cdot A$）是指电压与电流的有效值乘积。$S = V_{rms} I_{rms}$  
平均功率与视在功率的比值又被称为功率因数，$pf = \frac{P}{S} = \cos(\theta_v - \theta_i)$  
功率因数是电压与电流的相位角之差的余弦值，也是负载阻抗幅角的余弦值。

### 复功率
复功率（单位为$V \cdot A$）是电压相量有效值和电流相量有效值的共轭复数的乘积，它的值是一个复数，实部为有功功率P，虚部为无功功率Q。即：  
  - 复功率 $S = P + jQ = V_{rms} I_{rms} ^ {*} = |V_{rms}||I_{rms}|\angle(\theta_v - \theta_i)$
  - 视在功率 $S = |V_{rms}| |I_{rms}| = \sqrt{P^2 + Q^2}$ 亦为复功率之模
  - 有功功率 $P = Re(S) = S \cos(\theta_v - \theta_i)$
  - 无功功率 $Q = Im(S) = S \sin(\theta_v - \theta_i)$
  - 功率因子 $\frac{P}{S} = \cos(\theta_v - \theta_i)$
  
### 功率守恒
电源的复功率、有功功率、无功功率分别等于各个负载上的复功率、有功功率、无功功率之和。

### 功率因数的校正
不改变原始负载的电压或电流，提高功率因数的过程称为功率因数校正。

由于大多数负载都是感性的，所以给负载并联一个电容即可改善或校正负载的功率因数。在不改变有功功率的情况下，如果将功率因数从 $\cos \theta_1$ 提高到 $\cos \theta_2$，那么所需并联的电容容值C由下式确定：  
  - $C = \frac{Q_C}{\omega V_{rms} ^ {2}} = \frac{P(\tan \theta_1 - \tan \theta_2)}{\omega V_{rms} ^ {2}}$

若负载是容性的，则负载因串联一个电感器以实现功率因数校正，所需的分流电感器的值$L$可由下式确定：  
  - $L = \frac{V_{rms} ^ {2}}{\omega Q_L}$ 其中，$Q_L = Q_1 - Q_2$，表示新、旧无功功率之差

## 频率响应

### 频率响应的概念
电路的频率响应是指电路的行为特征随信号频率变化而发生的变化。

### 传递函数
电路的传递函数$H(\omega)$是随着频率而变化的输出相量$Y(\omega)$（元件的电压或电流）与输入相量$X(\omega)$（源电压或电流）之比。  
传递函数一般表示为： $H(\omega) = \frac{Y(\omega)}{X(\omega)}$

确定传递函数时，首先要将电路中的电阻器、电感器与电容器用它们的阻抗 $R$、$j \omega L$、$\frac{1}{j \omega C}$ 取代，得到频率等效电路，然后利用电路分析方法确定式子中的变量，绘制出传递函数的模与相位随频率变化的曲线。


