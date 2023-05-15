---
title: TSL1401驱动开发与状态机思想小议
date: 2023-05-10 12:59:07
tags: 
- MCU
- NUEDC
categories:
- MCU
- NUEDC
---

## TSL1401

### Why TSL1401?

校赛的时候由于连熬几个大夜，赛前忘了调红外阈值，开局跑飞。遂下定决心，扔掉红外，换一种循迹方案。正好老师买了TSL1401CL，于是便有了本文。

### 驱动时序

这是从TSL1401的DataSheet上截取的时序图:

<center>

![TSL1401TimingGeneral](https://s2.loli.net/2023/05/10/ilTJ2aVnItq83OB.png)

![TSL1401Timing](https://s2.loli.net/2023/05/10/qUwOrNz2KZ8QXIv.png)

</center>

可以看到，TSL1401的时序还是相对简单的，连续给129个CLK，在起止周期分别给SI脉冲就可以了。但实际上这里有一点坑，
曝光时间是固定的110个周期，如果要做自动曝光的话，就需要调节时钟频率来控制曝光时间。

### 第一版代码

由于TSL1401的时序并不复杂，我们可以直接用GPIO模拟时序。代码大概是这样:

```c
void TSL1401Initialize(void) {
    AverageBrightness = 0;
    memset(FrameBuffer, 0, 128);
    HAL_SYSTICK_Config(HAL_RCC_GetHCLKFreq() / 1000000);
}
uint16_t TSL1401ReadOut(uint16_t* buffer, uint16_t HalfClockPeriod) {
    uint32_t AverageBrightness = 0;
    HAL_GPIO_WritePin(TSL_CLK_GPIO_Port, TSL_CLK_Pin, GPIO_PIN_SET);
    HAL_GPIO_WritePin(TSL_SI_GPIO_Port, TSL_SI_Pin, GPIO_PIN_RESET);
    HAL_Delay(1);
    HAL_GPIO_WritePin(TSL_CLK_GPIO_Port, TSL_CLK_Pin, GPIO_PIN_RESET);
    HAL_GPIO_WritePin(TSL_SI_GPIO_Port, TSL_SI_Pin, GPIO_PIN_SET);
    HAL_Delay(1);
    HAL_GPIO_WritePin(TSL_CLK_GPIO_Port, TSL_CLK_Pin, GPIO_PIN_SET);
    HAL_GPIO_WritePin(TSL_SI_GPIO_Port, TSL_SI_Pin, GPIO_PIN_RESET);
    for(uint8_t i = 0; i < 128; i++) {
        HAL_GPIO_WritePin(TSL_CLK_GPIO_Port, TSL_CLK_Pin, GPIO_PIN_RESET);
        HAL_ADC_PollForConversion(&TSL_ADC_HANDLE, HAL_MAX_DELAY);
        buffer[i] = HAL_ADC_GetValue(&TSL_ADC_HANDLE);
        AverageBrightness = AverageBrightness + buffer[i];
        HAL_Delay(HalfClockPeriod);
        HAL_GPIO_WritePin(TSL_CLK_GPIO_Port, TSL_CLK_Pin, GPIO_PIN_SET);
    }
    AverageBrightness = AverageBrightness / 128;
    return AverageBrightness;
}

void TSL1401AutoExposure(void) {
    uint16_t avg = 0;
    uint16_t frameBuffer[128];
    uint16_t halfClockPeriod = 90;
    while ((avg < 1000 && halfClockPeriod < 500) || (avg > 3500 && halfClockPeriod > 5)) {
        avg = TSL1401ReadOut(frameBuffer);
        if (avg < 1000)
            halfClockPeriod = halfClockPeriod + 5;
        else if (avg > 3500)
            halfClockPeriod = halfClockPeriod - 5;
    }
}
```

写起来非常的轻松愉快，可谓是一气呵成。但就在我写完的一瞬间，我意识到了一个问题：这个程序是阻塞的。这也就意味着这个程序在执行的时候将把整个程序卡住。这玩意可是要整合上PID + 循迹来调小车的，这要是卡个100mS(最大曝光时间)我的PID不得飞到天上去，还是得重写一版非阻塞的驱动。

### 非阻塞版驱动

#### API设计

考虑到是非阻塞式的API，最简单的实现方式就是采用观察者模式，即当读取完成时通知。而在MCU裸机编程中，最简单的方式就是Callback，于是我们有了第一个API：

```c
TSL1401ReadCpltCallback()
```

那么参数和返回值该如何定义呢？Callback将由驱动调用,返回值在这个驱动中似乎是不必要的。而我们希望驱动能将读取完的FrameBuffer传递给外界，于是就定义成这样：

```c
void TSL1401ReadCpltCallback(uint16_t* frameBuffer);
```

同时，为了在默认情况下过编译，定义一个weak的实现：

```c
__attribute__((weak)) void TSL1401ReadCpltCallback(uint16_t* frameBuffer) {

}
```

其实这里还塞了其他的东西，暂且按下不表。  
由于这里还涉及了ADC读取的问题，既然都非阻塞了，那就全部非阻塞好了，ADC配置成中断模式，当ADC读取完成的时候通知一下驱动就好了，于是我们再设计一个回调服务函数，在ADC读取完成回调中调用：

```c
void __ADCCallbackService_TSL1401(ADC_HandleTypeDef* hadc);
```  

剩下的部分其实就比较常规了，无非是功能的启用禁用、曝光时间修改、触发单次曝光和初始化。依然由于是非阻塞的关系，我们需要做一个循环被调用的函数，不妨按OS编程的叫法，叫他Service，于是整体的API就出来了，大概是这样：

```c
void TSL1401ReadCpltCallback(uint16_t* frameBuffer);
void __ADCCallbackService_TSL1401(ADC_HandleTypeDef* hadc);
void TSL1401Initialize(void);
void TSL1401EnableAutoExposure(void);
void TSL1401DisableAutoExposure(void);
void TSL1401EnableContinouosExposure(void);
void TSL1401DisableContinouosExposure(void);
void TSL1401SetExposureTime(uint32_t exposureTime_uS);
void TSL1401BurstReadAsync(void);
void TSL1401ServiceFunction(void);
```

注意我给回调服务函数加了两个下划线作为前缀，这是笔者的代码习惯，通常只希望在内部或某个特定地点调用的函数或是变量，笔者都会加上这个前缀。

#### 驱动实现

笔者习惯于将一个c文件内部的变量pack成结构体(面向对象后遗症属于是)，先把变量定义下：

```c
struct {
    uint8_t     NextBurstReadFlag:1;
    uint8_t     AutoExposureEnable:1;
    uint8_t     AdcDataAvailableFlag:1;
    uint8_t     ContinuousExposureEnable:1;
    uint8_t     ExposureAndReadOutState:4;
    uint16_t    HalfClockPeroid;
    uint8_t     ClockCycleCounter;
    uint32_t    AverageBrightness;
    uint32_t    OperationStartTick;
} __TSL1401DriverVariablesPack = { 0 };

uint16_t __TSL1401FrameBuffer[128] = { 0 };
```

然后把只需要动配置的函数都实现了：

```c
__attribute__((weak)) void TSL1401ReadCpltCallback(uint16_t* frameBuffer) {

}

void __ADCCallbackService_TSL1401(ADC_HandleTypeDef *hadc) {
    if (hadc->Instance = TSL_ADC.Instance)
        __TSL1401DriverVariablesPack.AdcDataAvailableFlag = 1;
}

void TSL1401Initialize(void) {
    __TSL1401DriverVariablesPack.NextBurstReadFlag = 0;
    __TSL1401DriverVariablesPack.AutoExposureEnable = 1;
    __TSL1401DriverVariablesPack.AdcDataAvailableFlag = 0;
    __TSL1401DriverVariablesPack.ContinouosExposureEnable = 1;
    __TSL1401DriverVariablesPack.ExposureAndReadOutState = 0;
    __TSL1401DriverVariablesPack.HalfClockPeriod = 90;
    __TSL1401DriverVariablesPack.ClockCycleCounter = 0;
    __TSL1401DriverVariablesPack.AverageBrightness = 0;
    __TSL1401DriverVariablesPack.OperationStartTick = 0;
    HAL_SYSTICK_Config(HAL_RCC_GetHCLK() / 1000000);
}

void TSL1401EnableAutoExposure(void) { __TSL1401DriverVariablesPack.AutoExposureEnable = 1; }

void TSL1401DisableAutoExposure(void) { __TSL1401DriverVariablesPack.AutoExposureEnable = 0; }

void TSL1401EnableContinouosExposure(void) { __TSL1401DriverVariablesPack.ContinouosExposureEnable = 1; }

void TSL1401DisableContinouosExposure(void) { __TSL1401DriverVariablesPack.ContinouosExposureEnable = 0; }

void TSL1401SetExposureTime(uint32_t exposureTime_uS) { __TSL1401DriverVariablesPack.HalfClockPeriod = exposureTime_uS / 220; }

void TSL1401BurstReadAsync(void) {
    if (__TSL1401DriverVariablesPack.ContinouosExposureEnable)
        return;
    __TSL1401DriverVariablesPack.NextBurstReadFlag = 1;
}
```

值得注意的是笔者将SysTick配置到了1uS来提供时钟源(因为小车上的定时器不够用了)，这将导致HAL_Delay()的延时单位从mS变成uS，如果需要HAL_Delay()的读者需要自己配置一个1uS的tick来提供时钟源。  
接下来就是重头戏——Service的编写。其实从__TSL1401DriverVariablesPack的定义中就能看出我的意图——状态机。对着时序图切分状态，我们不妨定义一个枚举类型：

```c
typedef enum {
    SIPulseGenerate_ClkHighSILow_E  = 0x0,
    SIPulseGenerate_ClkLowSIHigh_E  = 0x1,
    ReadOut_ClkLowAdcStart_E        = 0x2,
    ReadOut_ClkLowAdcConvCplt_E     = 0x3,
    ReadOut_ClkHigh_E               = 0x4,
    ReadOut_AllDone_E               = 0x5,
    Idle_E                          = 0xF
} ExposureAndReadOutState_E;
```

然后把__TSL1401DriverVariablesPack改写成这样：

```c
struct {
    uint8_t     NextBurstReadFlag:1;
    uint8_t     AutoExposureEnable:1;
    uint8_t     AdcDataAvailableFlag:1;
    uint8_t     ContinuousExposureEnable:1;
    uint8_t     reservedBits:4;
    uint16_t    HalfClockPeroid;
    uint8_t     ClockCycleCounter;
    uint32_t    AverageBrightness;
    uint32_t    OperationStartTick;
    ExposureAndReadOutState_E   ExposureAndReadOutState;
} __TSL1401DriverVariablesPack = { 0 };

```

同时改写下初始化：

```c
void TSL1401Initialize(void) {
    __TSL1401DriverVariablesPack.NextBurstReadFlag = 0;
    __TSL1401DriverVariablesPack.AutoExposureEnable = 1;
    __TSL1401DriverVariablesPack.AdcDataAvailableFlag = 0;
    __TSL1401DriverVariablesPack.ContinouosExposureEnable = 1;
    __TSL1401DriverVariablesPack.ExposureAndReadOutState = SIPulseGenerate_ClkHighSILow_E;
    __TSL1401DriverVariablesPack.HalfClockPeriod = 90;
    __TSL1401DriverVariablesPack.ClockCycleCounter = 0;
    __TSL1401DriverVariablesPack.AverageBrightness = 0;
    __TSL1401DriverVariablesPack.OperationStartTick = 0;
    HAL_SYSTICK_Config(HAL_RCC_GetHCLK() / 1000000);
}
```

然后开始对着时序图画一个状态机：

![TSL1401State.png](https://s2.loli.net/2023/05/11/7GSWkgHxfDqu3y2.png)

接下来对着状态机写出函数即可：

```c
void TSL1401ServiceFunction(void) {
    switch(__TSL1401DriverVariablesPack.ExposureAndReadOutState) {
        case SIPulseGenerate_ClkHighSILow_E:
            if(__TSL1401DriverVariablesPack.ClockCycleCounter == 0) {
                if(HAL_GetTick() > __TSL1401DriverVariablesPack.OperationStartick + __TSL1401DriverVariablesPack.HalfClockPeriod){
                    HAL_GPIO_WritePin(TSL_CLK_GPIP_Port, TSL_CLK_Pin, GPIO_PIN_SET);
                    HAL_GPIO_WritePin(TSL_SI_GPIO_Port, TSL_SI_Pin, GPIO_PIN_RESET);
                    __TSL1401DriverVariablesPack.OperationStartick = HAL_GetTick();
                    __TSL1401DriverVariablesPack.ExposureAndReadOutState = SIPulseGenerate_ClkLowSIHigh_E;
                }
            } else {
                __TSL1401DriverVariablesPack.ExposureAndReadOutState = ReadOut_ClkLowAdcStart_E;
            }
            break;
        case SIPulseGenerate_ClkLowSIHigh_E:
            if(HAL_GetTick() > __TSL1401DriverVariablesPack.OperationStartick + __TSL1401DriverVariablesPack.HalfClockPeriod) {
                HAL_GPIO_WritePin(TSL_CLK_GPIP_Port, TSL_CLK_Pin, GPIO_PIN_RESET);
                HAL_GPIO_WritePin(TSL_SI_GPIO_Port, TSL_SI_Pin, GPIO_PIN_SET);
                __TSL1401DriverVariablesPack.ClockCycleCounter += 1;
                __TSL1401DriverVariablesPack.OperationStartick = HAL_GetTick();
                __TSL1401DriverVariablesPack = SIPulseGenerate_ClkHighSILow_E;
            }
            break;
        case ReadOut_ClkLowAdcStart_E:
            if(__TSL1401DriverVariablesPack.ClockCycleCounter < 129) {
                if(HAL_GetTick() > __TSL1401DriverVariablesPack.OperationStartick + __TSL1401DriverVariablesPack.HalfClockPeriod) {
                    HAL_GPIO_WritePin(TSL_CLK_GPIP_Port, TSL_CLK_Pin, GPIO_PIN_RESET);
                    __TSL1401DriverVariablesPack.AdcDataAvailableFlag = 0;
                    __TSL1401DriverVariablesPack.OperationStartTick = HAL_GetTick();
                    HAL_ADC_Start_IT(&TSL_ADC);
                    __TSL1401DriverVariablesPack.ExposureAndReadOutState = ReadOut_ClkLowAdcConvCplt_E;
                }
            } else {
                __TSL1401DriverVariablesPack.ExposureAndReadOutState = ReadOut_AllDone_E;
            }
            break;
        case ReadOut_ClkLowAdcConvCplt_E:
            __TSL1401DriverVariablesPack.AverageBrightness += HAL_ADC_GetValue(&TSL_ADC);
            __TSL1401FrameBuffer[__TSL1401DriverVariablesPack.ClockCycleCounter - 1] = HAL_ADC_GetValue(&TSL_ADC);
            __TSL1401DriverVariablesPack.ExposureAndReadOutState = ReadOut_ClkHigh_E;
            break;
        case ReadOut_ClkHigh_E:
            if(HAL_GetTick() > __TSL1401DriverVariablesPack.OperationStartick + __TSL1401DriverVariablesPack.HalfClockPeriod) {
                HAL_GPIO_WritePin(TSL_CLK_GPIP_Port, TSL_CLK_Pin, GPIO_PIN_SET);
                __TSL1401DriverVariablesPack.OperationStartTick = HAL_GetTick();
                __TSL1401DriverVariablesPack.ExposureAndReadOutState = ReadOut_ClkLowAdcStart_E;
            }
            break;
        case ReadOut_AllDone_E:
            if(__TSL1401DriverVariablesPack.AutoExposureEnable) {
                if(__TSL1401DriverVariablesPack.AverageBrightness < 1000 && __TSL1401DriverVariablesPack.HalfClockPeriod < 1000) {
                    __TSL1401DriverVariablesPack.HalfClockPeriod += 50;
                    __TSL1401DriverVariablesPack.ExposureAndReadOutState = SIPulseGenerate_ClkHighSILow_E;
                } else if (__TSL1401DriverVariablesPack.AverageBrightness > 3500 && __TSL1401DriverVariablesPack.HalfClockPeriod > 100) {
                    __TSL1401DriverVariablesPack.HalfClockPeriod -= 50;
                    __TSL1401DriverVariablesPack.ExposureAndReadOutState = SIPulseGenerate_ClkHighSILow_E;
                } else {
                    TSL1401ReadCpltCallback(__TSL1401FrameBuffer);
                    if(__TSL1401DriverVariablesPack.ContinouosExposureEnable) {
                        __TSL1401DriverVariablesPack.ExposureAndReadOutState = SIPulseGenerate_ClkHighSILow_E;
                    } else {
                        __TSL1401DriverVariablesPack.ExposureAndReadOutState = Idle_E;
                    }
                }
            } else {
                if(__TSL1401DriverVariablesPack.ContinouosExposureEnable) {
                    __TSL1401DriverVariablesPack.ExposureAndReadOutState = SIPulseGenerate_ClkHighSILow_E;
                } else {
                    __TSL1401DriverVariablesPack.ExposureAndReadOutState = Idle_E;
                }
            }
            break;
        case Idle_E:
            if(__TSL1401DriverVariablesPack.NextBurstReadFlag) {
                __TSL1401DriverVariablesPack.NextBurstReadFlag = 0;
                __TSL1401DriverVariablesPack.ExposureAndReadOutState = SIPulseGenerate_ClkHighSILow_E;
            }
            break;
    }
}
```

至此，我们的TSL1401驱动算是编写完毕了，进入接下来的话题——状态机。

## 状态机编程

### 什么是状态机

状态机是对于事务运行规则的一种抽象，对于状态机而言，有四大基础概念

- 状态：即系统的状态
- 事件：执行某个操作所需要的触发条件
- 动作：事件发生后执行的操作
- 转移：从一个状态切换到另一个状态

以上面的TSL1401驱动为例，转移图中的判断就是“事件”，而状态转移和Flag、配置变量就是“动作”。  
事实上，在FPGA中使用状态机编程更多，而这里之所以用了状态机，是因为我们希望整个系统是异步的，而使用了Visitor设计模式。在Visitor设计模式中，操作完成会触发一个“事件”，为了管理这种“事件”触发的“动作”，我们便需要引入状态机。

### Why State Machine?

前文提到，要异步就得状态机，所以为什么选择状态机可以换一个问法，为什么用异步编程？其实说来说去也无非一句话——最大化执行效率。如果使用传统的同步(Synchronous)编程，IO方法事实上是阻塞的，这也就意味着，在等待IO操作完成的过程之中，CPU是空转的，效率非常低下。事实上，在新版本的Cpp标准(C++ 20)里引入了co_await和co_async关键字，可以简单的实现异步操作，然而嵌入式编译器一般跟进标准非常慢....并且在嵌入式里使用C++并不是什么明智的决定(会带来更大的RAM和ROM资源开销且g++等c++编译器的行为并不稳定)，所以在大部分时候我们还是倾向于使用状态机。