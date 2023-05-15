---
title: 常见MCU功耗测试
date: 2023-05-15 12:56:06
tags:
    - MCU
categories:
    - MCU
    - 瞎折腾
---

![STM32F103RBTx.jpg](https://s2.loli.net/2023/05/15/ZyzI7oUNfAQ4lu6.jpg)

## 一些哔哔

去楼上实验室Van♂的时候学弟跟我提了一嘴STM32U575标称功耗的事，顺便展示了以下他参加众筹的合宙IoT Power，遂临时起意，测了以下手头有的一些常见MCU的功耗，于是有了本文

## 测试结果

首先声明，受时间和测试条件限制，我并不能测到非常精确的功耗数据，以下的数据仅有参考价值：

| MCU型号       | 板卡         | 备注                                                          | 电流消耗 |
|---------------|--------------|---------------------------------------------------------------|----------|
| ESP32-S       | NodeMCU      | 满主频 WiFi + BLE + CH340 外部晶振                            | 59.40mA  |
| ESP32-S3      | CORE-ESP32S3-A10 | 满主频 WiFi + BLE + CP2102 外部晶振                       | 114.2mA  |
| ESP8266       | unknown      | 满主频 WiFi + CH340 外部晶振                                  | 81.66mA  |
| ESP8266       | ESP-01F      | 满主频 WiFi 外部晶振                                          | 79.97mA  |
| GW1N-4        | TangNano-4K  | 流水灯程序                                                    | 57.88mA  |
| MSP430G2553   | LaunchPad    | 满主频 无外设                                                 | 108.4uA  |
| MSP432E401    | LaunchPad    | 满主频 ADC满速轮询 XDS110电源已断开                           | 8.198mA  |
| RP2040        | Pico         | 满主频 无外设                                                 | 22.17mA  |
| STM32F103RBTx | NUCLEO       | 已断开STLink电源,满主频 + ADC 未设置未使用IO为GPIO Analog     | 4.056mA  |
| STM32F103C8Tx | BluePill     | 满主频 + ADC + 全TIM + 双外部晶振 未设置未使用IO为GPIO Analog | 10.60mA  |
| STM32F407ZGTx | FK407M2      | 满主频 无外设 双外部晶振 未设置未使用IO为GPIO Analog          | 69.43mA  |
| STM32G474RETx | NUCLEO       | 满主频 无外设                                                 | 41.04mA  |
| STM32H723VGTx | Boring       | 满主频 无外设 双外部晶振 未设置未使用IO为GPIO Analog          | 39.58mA  |
| STM32H723VGTx | Boring       | 满主频 + 1.16寸LCD 双外部晶振 未设置未使用IO为GPIO Analog     | 152.66mA |
| STM32H743ZITx | NUCLEO       | V1板卡 无外设 未设置未使用IO为GPIO Analog                     | 83.89mA  |
| STM32H753     | OpenMV4 Plus | OpenMV 板卡功耗 对MCU功耗无参考价值 未开启LED                 | 210.6mA  |
| STM32H750VBTx | FK750M1      | 满主频 无外设 双外部晶振 未设置未使用IO为GPIO Analog          | 193.5mA  |
| STM32U575ZITx | NUCLEO       | 满主频 + DAC Normal Node 全速轮询 未设置未使用IO为GPIO Analog | 14.05mA  |
| TMS320F28379D | LaunchPad    | 官方Blink例程(据说外设全部初始化)                             | 193.9mA  |
| XC7A35T       | E-Element    | 仅供电3V3，LED + FPGA 晶振 + 流水灯程序                       | 49.53mA  |
| XC7Z020       | 小熊猫       | 5V供电 Cortex-A满速 + PL无程序                                | 123.6mA  |

## 测试照片 (多图)

![ESP32-S.jpg](https://s2.loli.net/2023/05/15/4yCwr2mh1iebWVN.jpg)

![MSP432E401Y.jpg](https://s2.loli.net/2023/05/15/Kv7izbsPYIXWex9.jpg)

![OpenMV4 Plus](https://s2.loli.net/2023/05/15/crNuChPLbYRoVmS.jpg)

![STM32F103C8Tx.jpg](https://s2.loli.net/2023/05/15/kaMgoF1Uj72dCJ6.jpg)

![STM32F103RBTx.jpg](https://s2.loli.net/2023/05/15/ZyzI7oUNfAQ4lu6.jpg)

![STM32F407ZGTx.jpg](https://s2.loli.net/2023/05/15/VrANhtSH72soaGM.jpg)

![STM32H723VGTx.jpg](https://s2.loli.net/2023/05/15/gvJ9WE5NLZf3lp2.jpg)

![TangNano-4K.jpg](https://s2.loli.net/2023/05/15/HMrqlXESQ5Of4Bh.jpg)

![XC7A35T.jpg](https://s2.loli.net/2023/05/15/GyE8tRwQ5pclqKS.jpg)

## 写在结尾

看起来低功耗的王者还是MSP430G2553（bushi），如果说在能用的情况下的低功耗，ST的STM32U575和TI的MSP432E401似乎不错。可惜前几天老师把MSPM0L1306收回去了，不然测试一下也是好的。C2000系列的功耗一开始吓我一跳，后来学弟告诉我TI的例程把所有外设都初始化了，不管例程用不用....顺便吐槽下TI的嵌入式软件。STM32整体表现还不错，但H750.....一枝独秀了属于是，电表倒转。一个令我惊喜的MCU是ESP32-S，对于一个启用了WiFi的MCU，我对他的预计功耗在100mA以上，没想到只有40mA上下，而且ESP8266和ESP32-S3功耗都显著高于它(或许是我记错里面是啥程序了？)回头再写个空程序测试一下。如果手头拿到新板子的话大概会更新吧(咕咕咕)....