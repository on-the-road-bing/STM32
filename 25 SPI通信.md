# SPI通信

## SPI协议简介

- SPI（Serial Peripheral Interface）是由Motorola公司开发的一种高速全双工的通信总线。被广泛地使用在ADC、LCD等设备与MCU间，要求通讯速率较高的场合。
- 四根通信线：SCK、MOSI、MISO、SS（Slave Select 从机选择）
- 高速，同步，全双工
- 支持总线挂载多设备（一主多从）

### SPI物理层

<img src="https://gitee.com/yb_wang6/typora_imgs/raw/master/img/image-20260904160229250.png" alt="image-20260904160229250" style="zoom:33%;" />

SPI通讯使用3条总线及片选线，3条总线分别为SCK、MOSI、MISO，片选线为SS（也叫NSS、CS）。

- 所有SPI设备的SCK、MOSI、MISO分别连在一起。
- 主机另外引出多条SS控制线，分别接到各从机的SS引脚。
- 输出引脚配置为推挽输出，输入引脚配置为浮空或上拉输入。

### 协议层（MSB或LSB先行）

与I2C的类似，SPI协议定义了通讯的起始和停止信号、数据有效性、时钟同步等环节。

**起始和停止信号**

- 起始条件：SS从高电平切换到低电平
- 终止条件：SS从低电平切换到高电平

<img src="https://gitee.com/yb_wang6/typora_imgs/raw/master/img/image-20260904160441421.png" alt="image-20260904160441421" style="zoom: 50%;" />

**数据有效性**

SPI每次数据传输可以8位或16位为单位，每次传输的单位数不受限制。

**SPI通讯模式**

SPI有两个可配置位，分别是CPOL时钟极性，CPHA时钟相位，每一位可以配置为1或0，总共组合起来就有模式0、模式1、模式2、模式3四种模式。

- 时钟极性CPOL：Clock Polarity，指SPI通讯设备处于空闲状态时，SCK信号线的电平信号\(即SPI通讯开始前、 NSS线为高电平时SCK的状态\)。CPO=0时， SCK在空闲状态时为低电平，CPOL=1时，则相反。
- 时钟相位CPHA：Clock Phase，指数据采样的时刻，当CPHA=0时，MOSI或MISO数据线上的信号将会在SCK时钟线的“奇数边沿”被采样。当CPHA=1时， 数据线在SCK的“偶数边沿”采样。

由CPOL及CPHA的不同状态，SPI分成了四种模式。实际中采用较多的是“模式0”与“模式3”。

<img src="https://gitee.com/yb_wang6/typora_imgs/raw/master/img/image-20260904171543860.png" alt="image-20260904171543860" style="zoom:50%;" />

<img src="https://gitee.com/yb_wang6/typora_imgs/raw/master/img/image-20260904170656598.png" alt="image-20260904170656598" style="zoom: 40%;" />

## SPI特性及架构

SPI外设简介

- SPI外设可用作通讯的主机及从机， 支持最高的SCK时钟频率为f\(pclk\)/2 \(STM32 F103芯片默认f\(pclk1\)为36MHz， f\(pclk2\)为72MHz\)
- 支持SPI协议的4种模式，
- 数据帧长度可设置为8位或16位， 
- 可设置数据MSB先行或LSB先行。

SPI架构

<img src="https://gitee.com/yb_wang6/typora_imgs/raw/master/img/image64.png" alt="图 24‑5 SPI架构图" style="zoom: 60%;" />

## SPI相关结构体和函数

```c
 typedef struct {
     uint32_t Mode;      	/*设置SPI的主/从机端模式 */
     uint32_t Direction; 	/*设置SPI的单双向模式 */
     uint32_t DataSize;  	/*设置SPI的数据帧长度，可选8/16位 */
     uint32_t CLKPolarity;	/*设置时钟极性CPOL，可选高/低电平*/
     uint32_t CLKPhase; 	/*设置时钟相位，可选奇/偶数边沿采样 */
     uint32_t NSS;       	/*设置NSS引脚由SPI硬件控制还是软件控制*/
     uint32_t BaudRatePrescaler; /*设置时钟分频因子，fpclk/分频数=fSCK */
     uint32_t FirstBit; 	/*设置MSB/LSB先行 */
     uint32_t TIMode;   	/*指定是否启用TI模式 */
     uint32_t CRCCalculation; 	/*指定是否启用CRC计算*/
     uint32_t CRCPolynomial;  	/*设置CRC校验的表达式*/
 } SPI_InitTypeDef;

//初始化函数
HAL_StatusTypeDef HAL_SPI_Init(SPI_HandleTypeDef *hspi)
__HAL_SPI_ENABLE(__HANDLE__)
```

## 实验：SPI读写FLASH

1. 初始化通讯使用的目标引脚及端口时钟；
2. 使能SPI外设的时钟；
3. 配置SPI外设的模式、地址、速率等参数并使能SPI外设；
4. 编写基本SPI按字节收发的函数；
5. 编写对FLASH擦除及读写操作的的函数；
6. 编写测试程序，对读写数据进行校验。