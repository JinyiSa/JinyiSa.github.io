---
title: STM32U5踩坑——模拟外设篇
date: 2023-04-18 11:38:23
tags:
    - EE
    - 踩坑
    - STM32
categories:
    - STM32
    - 折腾记录
---
# 初识STM32U5
## 关于STM32U5
STM32U5系列是ST新推出的Cortex-M33架构的MCU产品线，主打超低功耗。由于架构的优化，Cortex-M33事实上也是一种高性能的架构。听起来好像很不错的样子，但是, there are no silver bullet。兼顾超低功耗和高性能必然带来一个结果——编程繁琐且容易出错。

## STM32U5的电源域与时钟树
### 电源域
对于一个现代的MCU而言，[电源域](https://en.wikipedia.org/wiki/Power_domains)的分割是必要的。一方面是某些模块对于供电有特殊要求，例如ADC、DAC和比较器这种模拟外设需要尽可能"纯净"的电源，把它们和数字部分的电源划分在一起显然是不合适的。另一方面是不同模块之间的电压不尽相同，例如在STM32U5中，$V_{IOBank}$需要3V3，而$V_{Core}$却只能接受1V8。U5的电源大致有这么几个域:  

- Core Domain
- $V_{DD}$ Domain
- Backup Domain ($V_{BAT}$)
- Analog Domain ($V_{DDA}$)
- SMPS Power Stage (这个电源域只有内置DC-DC的型号有)
- $V_{DDIO2}$ Domain
- $V_{DDUSB}$ Domain

看名称应该也能猜出来各个电源域负责什么东西，我们踩的一个大坑就和Power Domain有关，暂且按下不表。
### 时钟树
在一个规模稍大的数字系统中，一个时钟往往是不够用的。而我们不希望各个时钟都需要一个独立的震荡器，因为这显然过于消耗IO了。于是，我们在片内做上一系列的PLL，通过它们把输入的主时钟通过分频和倍频调整到合适的频率送给各个模块。我们将片内的PLL和MUX结构称为时钟树。这是从STM32U575[手册](https://www.semiee.com/file/ST/ST-STM32U575QI.pdf)中截取的时钟树示意图:  
![STM32U5ClockTree.png](https://s2.loli.net/2023/04/18/tJjUHSofqunCbOY.png)  
可以看到，U5的时钟树比较复杂。我们本次踩坑涉及的部分主要是DAC的时钟问题，这个留到下文再说。

## 点个灯吧!
讲了这么多，怎么能少了电子界的Hello World——Blink呢？由于笔者使用的NUCLEO-U575ZI-Q，CubeMX自带了BSP，笔者就直接从Board Selector中生成工程了。 其实直接使用默认配置即可，但笔者很不喜欢不开ICache的时候生成代码弹出的那个Warning，遂启用ICache:  
![NUCLEO-U575-ZI-QBlinkCubeMX.png](https://s2.loli.net/2023/04/18/HlnJYDUKMVOeuWp.png)
(反正ICache也没有啥需要配置的东西还能增强性能，何乐而不为呢)  
然后生成代码，打开工程，在初始化代码里加上:  
```c
uint8_t i = 0;
HAL_GPIO_WritePin(LED_RED_GPIO_Port, LED_RED_Pin, GPIO_PIN_SET);
HAL_GPIO_WritePin(LED_GREEN_GPIO_Port, LED_GREEN_Pin, GPIO_PIN_RESET);
HAL_GPIO_WritePin(LED_BLUE_GPIO_Port, LED_BLUE_Pin, GPIO_PIN_RESET);
```
然后在循环里加上:
```c
switch (i) {
    case 0:
        i++;
        HAL_GPIO_WritePin(LED_RED_GPIO_Port, LED_RED_Pin, GPIO_PIN_SET);
        HAL_GPIO_WritePin(LED_GREEN_GPIO_Port, LED_GREEN_Pin, GPIO_PIN_RESET);
        HAL_GPIO_WritePin(LED_BLUE_GPIO_Port, LED_BLUE_Pin, GPIO_PIN_RESET);
        break;
    case 1:
        i++;
        HAL_GPIO_WritePin(LED_RED_GPIO_Port, LED_RED_Pin, GPIO_PIN_RESET);
        HAL_GPIO_WritePin(LED_GREEN_GPIO_Port, LED_GREEN_Pin, GPIO_PIN_SET);
        HAL_GPIO_WritePin(LED_BLUE_GPIO_Port, LED_BLUE_Pin, GPIO_PIN_RESET);
        break;
    case 2:
        i++;
        HAL_GPIO_WritePin(LED_RED_GPIO_Port, LED_RED_Pin, GPIO_PIN_RESET);
        HAL_GPIO_WritePin(LED_GREEN_GPIO_Port, LED_GREEN_Pin, GPIO_PIN_RESET);
        HAL_GPIO_WritePin(LED_BLUE_GPIO_Port, LED_BLUE_Pin, GPIO_PIN_SET);
        break;
    case 3:
        i++;
        HAL_GPIO_WritePin(LED_RED_GPIO_Port, LED_RED_Pin, GPIO_PIN_RESET);
        HAL_GPIO_WritePin(LED_GREEN_GPIO_Port, LED_GREEN_Pin, GPIO_PIN_SET);
        HAL_GPIO_WritePin(LED_BLUE_GPIO_Port, LED_BLUE_Pin, GPIO_PIN_RESET);
        break;
    case 4:
        i = 0;
        HAL_GPIO_WritePin(LED_RED_GPIO_Port, LED_RED_Pin, GPIO_PIN_SET);
        HAL_GPIO_WritePin(LED_GREEN_GPIO_Port, LED_GREEN_Pin, GPIO_PIN_RESET);
        HAL_GPIO_WritePin(LED_BLUE_GPIO_Port, LED_BLUE_Pin, GPIO_PIN_RESET);
        break;
    default:
        break;
}
HAL_Delay(200);
```
然后直接编译下载即可。

# 玩玩模拟外设
STM32U575的模拟外设还是挺强悍的ADC1 14Bit 2.5Msps，ADC2 12Bit 2.5Msps。DAC两个通道12Bit，带可开启的SHA。但是有个学弟说U5的ADC有问题，怎么采都是0，笔者遂决定调一下ADC，用默认生成的代码，加上轮询，果然是0，于是就有了本文23333。
## ADC调试
我先是按其他系列的配置方式把ADC配置为轮询单次采样，CubeMX生成的代码大概是这样：
```c
void MX_ADC1_Init(void){
/* USER CODE BEGIN ADC1_Init 0 */
/* USER CODE END ADC1_Init 0 */
/* USER CODE BEGIN ADC1_Init 1 */
/* USER CODE END ADC1_Init 1 */
/** Common config*/
 hadc1.Instance = ADC1;
 hadc1.Init.ClockPrescaler = ADC_CLOCK_ASYNC_DIV1;
 hadc1.Init.Resolution = ADC_RESOLUTION_14B;
 hadc1.Init.GainCompensation =   0;
 hadc1.Init.ScanConvMode = ADC_SCAN_DISABLE;
 hadc1.Init.EOCSelection = ADC_EOC_SINGLE_CONV;
 hadc1.Init.LowPowerAutoWait = DISABLE;
 hadc1.Init.ContinuousConvMode = DISABLE;
 hadc1.Init.NbrOfConversion =   1;
 hadc1.Init.DiscontinuousConvMode = DISABLE;
 hadc1.Init.DMAContinuousRequests = DISABLE;
 hadc1.Init.TriggerFrequencyMode = ADC_TRIGGER_FREQ_HIGH;
 hadc1.Init.Overrun = ADC_OVR_DATA_PRESERVED;
 hadc1.Init.LeftBitShift = ADC_LEFTBITSHIFT_NONE;
 hadc1.Init.ConversionDataManagement = ADC_CONVERSIONDATA_DR;
 hadc1.Init.OversamplingMode = DISABLE;
 if   (HAL_ADC_Init(&hadc1) != HAL_OK) {
 Error_Handler();
 }
/* USER CODE BEGIN ADC1_Init 2 */
/* USER CODE END ADC1_Init 2 */
}
```
看上去很合理对吧，我们在主程序里添加轮询代码：
```c
HAL_ADC_Start(&hadc1);
volatile uint16_t adcValue;
volatile uint16_t retValue;
while(1) {
    retValue = HAL_ADC_PollForConversion(&hadc1, 50);
    adcValue = HAL_ADC_GetValue(&hadc1);
}
``` 
这里随手定义的retValue在后面的调试给我帮了大忙，后面会提到。不如直接开始编译调试吧，然后我就得到了这样的结果:    
![Debug_ScreenShot.png](https://s2.loli.net/2023/04/18/1fJIWxKoM6iHOrk.png)  
retValue是3，adcValue是0。retValue不是HAL_OK (也就是0)，这中间肯定出了问题，先找找3是什么意思吧。在一番查找之后，可以在stm32u5xx_hal_def.h中找到：
```c
typedef enum {
    HAL_OK = 0x00,
    HAL_ERROR = 0x01,
    HAL_BUSY = 0x02,
    HAL_TIMEOUT = 0x03
} HAL_StatusTypeDef;
```
retValue是3，这意味着Time Out。但是50mS的时间怎么都该足够让一个2.5Msps的ADC完成一次采样了，百思不得其解。遂找一调过U5的学弟询问此事，学弟答：需要在初始化里加入如下代码：
```c
  SET_BIT(RCC->AHB3ENR, RCC_AHB3ENR_PWREN);
  HAL_Delay(1);
  SET_BIT(PWR->SVMCR, PWR_SVMCR_AVM1EN);
  HAL_Delay(1);
  while (READ_BIT(PWR->SVMSR, PWR_SVMSR_VDDA1RDY) == 0); 
  SET_BIT(PWR->SVMCR, PWR_SVMCR_ASV); 
  CLEAR_BIT(RCC->AHB3ENR, RCC_AHB3ENR_PWREN);
```
怎么是手动操作寄存器，赣！遂在工程中一个一个搜索Mask，功夫不负有心人。我在stm32u5xx_hal_pwr_ex.c中找到了这样一个函数：
```c
/**
* @brief Enable VDDA supply.
* @note Remove VDDA electrical and logical isolation, once VDDA supply is
* present for consumption saving.
* @retval None.
*/
void HAL_PWREx_EnableVddA(void) {
 SET_BIT(PWR->SVMCR, PWR_SVMCR_ASV);
}
```
似乎是启用$V_{DDA}$电源域的函数。笔者的开发环境有调用计数功能，我一看函数上面赫然显示"0 ref"，好家伙，合着您压根没开$V_{DDA}$电源域是吗。遂在MX_ADC1_Init中加上启用电源域的代码：
```c
HAL_PWREx_EnableVddA();
```
然后重新编译调试，果然ADC能够正常采样了。所以....我又跑回去仔细检查了CubeMX配置页面，发现根门没有电源域配置这一项。所以.....即使U5已经上市这么久了，CubeMX的支持还是个半成品......ST你坏事做尽！

## DAC调试
虽然学弟提的ADC问题解决了，但那个调过U5的学弟又说了另外一个问题，它的DAC似乎不太对，只能运行在很低的采样率下。我想着反正已经开始调了，不如一鼓作气把DAC也调一下。我没有急着开始Coding，鉴于学弟说速度很低，我首先想到的是时钟树配置问题，我在时钟树里找到了这个：
![DAC_Sample_Hold_Clock_MUX.png](https://s2.loli.net/2023/04/18/ev8QUjgcupkVOmG.png)  
我打出一个？，怎么会接在LSI或者LSE上的，难道DAC只能低频工作吗？我总觉得不太对劲，遂打开了U5的Reference Manual找到DAC部分的DAC Features表格：  
![U5_DAC_Features.png](https://s2.loli.net/2023/04/18/7Z9gSKjDL2oqiT4.png)
Maximum sampling rate（吐槽一下，ST怎么把采样率写成采样时间了）1Msps，我现在一头雾水，32.768KHz的Clk怎么能有1Msps的采样率，遂继续读RM，我先是看到了在DAC conversion里这样一段话:  
HFSEL bits of DAC_MCR must be set when dac_hclk or dac_ker_ck clock speed is faster than 80 MHz. It adds an extra delay to the transfer from DAC_DHRx register to DAC_DORx register. Refer to Table HFSEL description below for the limitation of the DAC_DORx update rate depending on HFSEL bits and dac_hclk clock frequency.If the data is updated or a software/hardware trigger event occurs during the non-allowed.period, the peripheral behavior is unpredictable.The above timing is only related to the limitation of the DAC interface. Refer also to the tSETTLING parameter value in the product datasheet.  
似乎这个参数我也没配置正确来着，但这不是主要问题，我仍不知道1Msps的采样率从何而来，遂继续看，终于在DAC channel modes一节中找到了问题的答案。U5的DAC分为Normal mode和Sample and hold mode。我找到的那个MUX只负责SHA。。。。破案了。
于是回到CubeMX，配置好DAC High Frequency Mode和Mode Select，生成代码，写一个打锯齿波的程序:
```c
uint16_t i = 0;
HAL_DAC_Start(&hdac1, DAC_CHANNEL_1);
while(1) {
    HAL_DAC_SetValue(&hdac1, DAC_CHANNEL_1, DAC_ALIGN_12B_R, i);
    i = i < 4096 ? i + 1 : 0;
}
```
编译下载，然后挂上示波器，得到了一个大约2KHz的锯齿波。一个周期4096个采样点，换算出来实际采样率居然达到了惊人的4Msps。看来ST的手册还是过于保守了，强行轮询都能做到4Msps，手册上居然只标注为1Msps...…

