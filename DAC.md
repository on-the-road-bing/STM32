# DAC

数字模拟转换器（DAC，Digital-to-Analog Converter）是微控制器中用于将数字信号转换为模拟信号的重要外设。STM32微控制器内置了DAC模块，允许开发者将数字值转换为相应的模拟电压输出。这在音频处理、电机控制、信号生成等应用中非常有用。

<img src="https://gitee.com/yb_wang6/typora_imgs/raw/master/img/image-20261102200926121.png" alt="image-20261102200926121" style="zoom: 50%;" />

## DAC的工作原理

DAC的核心功能是将数字信号转换为模拟信号。STM32的DAC模块通常支持12位分辨率，这意味着它可以输出4096个不同的电压级别（从0到4095）。DAC的输出电压范围通常为0V到参考电压（Vref），Vref可以是内部参考电压或外部提供的参考电压。

### DAC的基本结构

STM32的DAC模块通常包含以下部分：

-  数据寄存器：用于存储待转换的数字值。
- DAC输出缓冲器：用于驱动外部负载。
- 触发源：用于控制DAC转换的时机，可以是定时器、外部事件等。

![image-20261102201545442](https://gitee.com/yb_wang6/typora_imgs/raw/master/img/image-20261102201545442.png)

## 配置STM32 DAC

1. 启用DAC时钟：首先需要启用DAC外设的时钟。
2. 配置GPIO引脚：将DAC输出引脚配置为模拟模式。
3. 初始化DAC：设置DAC的分辨率、触发源等参数。
4. 启动DAC：使能DAC并开始转换。

```c
DAC_HandleTypeDef hdac;

void DAC_Init(void) {
    // 启用DAC时钟
    __HAL_RCC_DAC_CLK_ENABLE();

    // 配置DAC输出引脚为模拟模式
    GPIO_InitTypeDef GPIO_InitStruct = {0};
    GPIO_InitStruct.Pin = GPIO_PIN_4; // 假设使用PA4作为DAC输出
    GPIO_InitStruct.Mode = GPIO_MODE_ANALOG;
    GPIO_InitStruct.Pull = GPIO_NOPULL;
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

    // 初始化DAC
    hdac.Instance = DAC;
    HAL_DAC_Init(&hdac);

    // 配置DAC通道1
    DAC_ChannelConfTypeDef sConfig = {0};
    sConfig.DAC_Trigger = DAC_TRIGGER_NONE; // 不使用外部触发
    sConfig.DAC_OutputBuffer = DAC_OUTPUTBUFFER_ENABLE; // 启用输出缓冲器
    HAL_DAC_ConfigChannel(&hdac, &sConfig, DAC_CHANNEL_1);

    // 启动DAC
    HAL_DAC_Start(&hdac, DAC_CHANNEL_1);
}

void Set_DAC_Value(uint32_t value) {
    // 设置DAC输出值
    HAL_DAC_SetValue(&hdac, DAC_CHANNEL_1, DAC_ALIGN_12B_R, value);
}
```



## 实验

1. 音频信号生成（还原）

   DAC可以用于生成音频信号。例如，通过将音频样本数据（如WAV文件中的采样值）连续写入DAC，可以生成音频输出。

2. 电机控制

   在电机控制中，DAC可以用于生成PWM信号的参考电压，从而控制电机的转速。通过调整DAC的输出电压，可以改变PWM信号的占空比，进而控制电机的转速。