---
title: 电赛训练笔记——电容测量篇
date: 2023-04-23 17:30:53
tags: 
- EE
- 模电
- STM32
- NUEDC
categories: NUEDC
---
## 题目要求

1. 使用单片机UART或LCD屏显示测量结果
2. 测量范围10nF~100nF
3. 测量绝对误差不大于1nF

## 参考方案分析

老师给我们提供了一个参考方案——用555搭建震荡器，通过测555输出频率来求解电容量。虽然笔者并不打算用这个方案，但还是分析一下好了 ~~(不然怎么能体现出我方案的优越性呢)~~。这个方案的电路大概长这样:  
![NE555_Osc_Circuit.png](https://s2.loli.net/2023/04/23/YOPqazVRyGXs18i.png)
<center>
(电路图来源于网络，侵删)
</center>
讲道理，这玩意简单到我懒得分析原理了，但不分析原理好像又不太能讲明白为啥不用....  
简单分析一下好了。首先，555的内部结构长这样:  

![NE555_Internal_Circuit.png](https://s2.loli.net/2023/04/23/95dUvxmbYSjeRKn.png)

如果把这玩意替换到上面的图里，555无稳态振荡器的原理就很明了了：

- Stage1: 上电初期，RS触发器的输出为0，DISCH和GND间的BJT关断，此时 $V_{cc}$ 通过 $R_{1}$和$R_{2}$给C充电
- Stage2: 充电至$U_{C} > \frac{1}{3}V_{cc}$, 此时，下面的比较器输出低电平，S端被置0，但R端依然为0，RS触发器状态不变
- Stage3: 充电至$U_{C} > \frac{2}{3}V_{cc}$，此时，上面的比较器输出高电平，R端被置1，而S端在Stage2中就成了0，RS触发器的输出为1，此时DISCH和GND间BJT导通，电容通过$R_{2}$放电
- Stage4: 放电至$U_{C} < \frac{2}{3}V_{cc}$，此时，上面的比较器输出低电平，R端被置0，此时S依然为0，RS触发器输出仍然为1
- Stage5: 放电至$U_{C} < \frac{1}{3}V_{cc}$，此时，下面的比较器输出高电平，S端被置1，此时RS触发器状态变化，输出翻转到0，状态回到Stage1

根据上面的过程，我们可以很容易的计算NE555的输出频率和占空比。由于稳定下来之后$U_{C}$一直介于$\frac{1}{3}V_{cc}$和$\frac{2}{3}V_{cc}$之间，对于充电过程，有：
$$ t_{charge} = (R_{1} + R_{2})C\ln{\frac{\frac{2}{3}V_{cc} - V_{cc}}{\frac{1}{3}V_{cc} - V_{cc}}} $$
$$ t_{charge} = (R_{1} + R_{2})C\ln{2} $$
同理可得：
$$ t_{discharge} = R_{2}C\ln{2} $$
所以有：
$$ f = \frac{1}{t_{charge} + t_{discharge}} = \frac{1}{(R_{1} + R_{2})C\ln{2} + R_{2}C\ln{2}}$$
$$ DutyCycle = \frac{t_{charge}}{t_{discharge} + t_{charge}} = \frac{(R_{1} + R_{2})C\ln{2}}{(R_{1} + R_{2})C\ln{2} + R_{2}C\ln{2}} = \frac{R_{1} + R_{2}}{R_{1} + 2R_{2}}$$

Seems good! 然而这个电路有其不可避免的问题：
- 首先，NE555本身飘的很厉害。
- 其次，这个电路里出现了$\ln{2}$，这是个无理数。在处理数据时必然存在浮点数精度丢失的问题。

于是——就引出了我的方案：V-I变换器恒流充电 + STM32模拟看门狗

## 我的方案

### 硬件设计 (V-I变换器)

先设计好V-I变换器的拓扑，笔者使用双反馈结构的V-I变换器，电路如下:

![V-IConverterWithTrim.png](https://s2.loli.net/2023/04/26/fEwkUAyNVTbt369.png)

事实上，这个电路依然非常简单。$V_{Trim}$是用来调零的信号，在分析阶段我们可以认为它就是0（不知道什么是调零的[戳这里](https://wiki.analog.com/university/courses/electronics/text/chapter-3)）。我们先约定一些记号：

| 符号           | 含义                    |
|----------------|-----------------------|
| $U_{in}$       | $V_{input+}$ 节点电压   |
| $U_{1}A_{-}$   | $U_{1}A$ 反相端电压     |
| $U_{1}A_{+}$   | $U_{1}A$ 同相端电压     |
| $U_{1}A_{Out}$ | $U_{1}A$ 输出电压       |
| $U_{1}B_{+}$   | $U_{1}B$ 同相端电压     |
| $U_{out}$      | $V_{output}$ 节点电压   |
| $I_{out}$      | $R_{5}$和$C_{3}$ 的电流 |

对于这个电路，由运放定律可知,在稳态下，必然有： 

$$ \begin{equation}
U_{1}A_{+} = U_{1}A_{-} 
\end{equation} $$

我们分析 $U_{1}A_{-}$，很容易有(电阻分压)：

$$ \begin{equation}
U_{1}A_{-} = U_{1}A_{Out} \times \frac{R_2}{R_2 + R_4}
\end{equation} $$

然后看 $U_{1}A_{+}$，同理有：

$$ \begin{equation}
U_{1}A_{+} = \left(U_{1}B_{Out} - U_{in}\right) \times \frac{R_1}{R_1 + R_3}
\end{equation} $$

由于 $U_{1}B$ 是跟随器，有：

$$ \begin{equation}
U_{1}B_{Out} = U_{Out}
\end{equation} $$

代回式(3)，有：

$$ \begin{equation}
U_{1}A_{+} = \left(U_{Out} - U_{in}\right) \times \frac{R_1}{R_1 + R_3}
\end{equation} $$

联立式(1)(2)(5)，有：

$$ \begin{equation}
U_{1}A_{Out} \times \frac{R_2}{R_2 + R_4} = \left( U_{Out} - U_{in} \right) \times \frac{R_1}{R_1 + R_3}
\end{equation} $$

对于我们的电路而言：

$$
R_1 = R_2 = R_3 = R_4 = 1K\Omega
$$

故而，式(6)稍作整理有：

$$ \begin{equation}
U_{1}A_{Out} = U_{out} - U_{in}
\end{equation} $$

对于$R_5$有：

$$ \begin{equation}
I_{out} = \frac{U}{I} = \frac{U_{1}A_{Out} - U_{out}}{R_5}
\end{equation} $$

(7)带入(8)，有：
$$
I_{Out} = \frac{U_{in}}{R_5}
$$

至此，V-I变换器的理论计算算是完成了，接下来就是$R_5$取值问题。这个电路的$V_{input}$将连接到STM32F407的DAC上，故而$U_{in} \in \left[ 0, 3.3 \right] V$ 。对于本题，笔者并不满足于题目的要求，笔者希望电路至少能测量1nF~1uF的电容，并且充电时间介于1mS和60mS之间(充电时间限制来自于软件，后文会提到)。很容易得出充电电流应该在 1uA到55uA之间。不妨取3.3V时满幅输出为60mA，则$R_5$应为$55K\Omega$，在$E_{24}$ 系列电阻中取最接近的 $56K\Omega$，满幅输出约为58.93uA，仍然满足要求。在输出为1uA时，DAC输出是560mV不算过小。所以，$R_5$取$56K\Omega$是合适的。  
至此，我们的硬件设计完成了。接下来就是Happy Coding。

### 软件设计

软件部分相对简单，直接给出设计思路好了。开启一个TIM提供1uS的定时中断，并设置一个累加器记录从充电开始到充电结束的时间。ADC开启模拟看门狗，上界设置为4095，这样在ADC输入达到3.3V时将触发模拟看门狗中断，在这个中断里关闭TIM的定时中断，并通过累加器的值计算电容量。

