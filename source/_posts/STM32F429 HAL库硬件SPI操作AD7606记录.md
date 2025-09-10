---
tags:
  - SPI
  - ADC
  - HAL库
  - f429
  - stm32
title: STM32F429 HAL库硬件IIC操作AD7606记录
excerpt: hal库操作AD7606模数转换模块的软硬件编程记录
categories: [嵌入式, 单片机, STM32]
date: 2025-09-10 21:51:35
---
## 分类介绍

AD7606是用于监测电压的ADC，16位，采样率200ksps。根据通道数可以分为3类

1.AD7606：可以接收8路电压输入
![image.png](https://fuyunyou-note.oss-cn-wuhan-lr.aliyuncs.com/typora-user-images/202508191833071.png)

2.AD7606-6：可以接收6路电压输入
![image.png](https://fuyunyou-note.oss-cn-wuhan-lr.aliyuncs.com/typora-user-images/202508191833581.png)

3.AD7606-4：接收4路电压输入
![image.png](https://fuyunyou-note.oss-cn-wuhan-lr.aliyuncs.com/typora-user-images/202508191832067.png)

## 硬件接线

![image.png](https://fuyunyou-note.oss-cn-wuhan-lr.aliyuncs.com/typora-user-images/202508301240700.png)

## 引脚介绍

### convstA和convstB

类型：数字输入引脚，
功能：AD7606可以接收多路电压输入，所有的电压输入被分为A和B两组，这两个引脚可以接收来自mcu的电平信号用于控制何时开始对应组的电压转换。

正常状态下向该引脚输入高电平，然后拉低一段时间后重新拉高表示开始转换。只需要保持极短时间的低电平即可，实测168MHz的f429在两次电平切换语句中不必延时。

这两个引脚还用于控制数据输出引脚（此处只介绍串行数据输出）[[#DOUTA和 DOUTB]]。
若分别单独拉低covA和covB，中间间隔一段时间，则数据分别从DoutA，DoutB输出。这种情况就需要两个引脚来接受数据。
可以在电路设计时将covA和covB连接在一起，电平同时变化，此时，所有通道的数据会从DoutA引脚输出。这样，就可以使用标准的SPI引脚的MOSI连接DoutA，在后续编程中可以很方便的使用SPI。

### PAR/SER/BYTE SEL

类型：数字输入引脚
功能：用于选择数据的串并行输出
一般选串行输出（能用SPI），直接接高电平即可。

### BUSY

类型：数字输出引脚
功能：MCU可以接收来自该引脚的数据，高电平表示AD处于busy状态正在进行转换。
可将该引脚接入MCU的一个IO，并设置为外部中断，下降沿触发。当中断发生，表明数据转完成，下一步就可以读数据进行处理了。
AD7606可以在采集时读数，也就是中断设为上升沿触发，按道理说采集时读出的数据应该是无效的，不知道这样设计有什么意义，但我没有实际测试过。

完整的一次通信过程（并行）：
![image.png](https://fuyunyou-note.oss-cn-wuhan-lr.aliyuncs.com/typora-user-images/202508191829653.png)
并行和串行只是SPI时序不同，触发SPI之前的时序是一样的。

### RANGE

类型：数字输入引脚
功能：模拟输入范围选择。
逻辑输入引脚。该引脚上的极性决定了模拟输入通道的输入范围。
连接到逻辑高电平，则所有通道的模拟输入范围为±10 V。
连接到逻辑低电平，则所有通道的模拟输入范围为±5 V。
该引脚上的逻辑变化会立即影响模拟输入范围。对于快速吞吐率应用，不建议在转换期间更改此引脚。

### FRSTDATA

类型：数字输出引脚
FRSTDATA输出信号指示第一个通道V1何时在并行、字节或串行接口上被输出。当CS输入为高电平时，FRSTDATA输出引脚处于三态。CS的下降沿使FRSTDATA脱离三态。
在并行模式下，与V1结果对应的RD下降沿会将FRSTDATA引脚置为高电平，表明V1的结果在输出数据总线上可用。FRSTDATA输出在RD的下一个下降沿后返回逻辑低电平。
在串行模式下，FRSTDATA在CS的下降沿变为高电平，因为这会将V1的最高有效位（MSB）通过DOUTA时钟输出。它在CS下降沿后的第16个SCLK下降沿返回低电平。

### DOUTA和 DOUTB

类型：数字输出引脚
功能：串行输出采集的数据
对于AD7606，从通道V1到通道V4的转换结果首先出现在DOUTA上，从通道V5到通道V8的转换结果首先出现在DOUTB上。
对于AD7606-6，从通道V1到通道V3的转换结果首先出现在DOUTA上，从通道V4到通道V6的转换结果首先出现在DOUTB上。
对于AD7606-4，通道V1和通道V2的转换结果首先出现在DOUTA上，通道V3和通道V4的转换结果首先出现在DOUTB上。

### CS和SCLK

时钟和片选和SPI同理
配置时：CPHA=1，CPOL=1
数据手册显示，支持的SPI最高通信速率为23.5MHz，但没必要跑这么快。
![image.png](https://fuyunyou-note.oss-cn-wuhan-lr.aliyuncs.com/typora-user-images/202508301232498.png)
cubeMX中配置如下所示，实测可以跑通
![image.png](https://fuyunyou-note.oss-cn-wuhan-lr.aliyuncs.com/typora-user-images/202508301232413.png)

器件的SPI通信时序
![image.png](https://fuyunyou-note.oss-cn-wuhan-lr.aliyuncs.com/typora-user-images/202508191851659.png)

### OS0、OS1和OS2

类型：数字输入引脚
功能：控制AD7606的数字滤波器的过采样率
![image.png](https://fuyunyou-note.oss-cn-wuhan-lr.aliyuncs.com/typora-user-images/202508301310880.png)
OS引脚在BUSY下降沿锁存，在此之前设置好即可。

## 软件编程

由于使用硬件SPI接受很简单，只需要在HAL库中点点就好了，如果不会看一下别人的教程，所以重点在如何触发SPI。

### 基本思路

1.拉低convst引脚，触发转换，7606会自动开始转换，同时拉高busy引脚
2.等待busy下降沿触发外部中断。在中断函数中设置标志量，然后在主函数中轮询。
3.检测到标志量被置位后，拉低CS引脚开启SPI通信，自动获取数据，然后处理。

### SPI配置

在cubeMX中点点就好了，注意几个要点
1.时钟相位和极性都是1
2.传输字长是16bit
![image.png](https://fuyunyou-note.oss-cn-wuhan-lr.aliyuncs.com/typora-user-images/202508301400154.png)

### 软件代码

```c
/*ad7606.c*/

//AD7606引脚配置
#define AD7606_FRST_Pin GPIO_PIN_9
#define AD7606_FRST_GPIO_Port GPIOE
#define AD7606_BUSY_Pin GPIO_PIN_10
#define AD7606_BUSY_GPIO_Port GPIOE
#define AD7606_BUSY_EXTI_IRQn EXTI15_10_IRQn
#define AD7606_CSN_Pin GPIO_PIN_11
#define AD7606_CSN_GPIO_Port GPIOE
#define AD7606_RESET_Pin GPIO_PIN_12
#define AD7606_RESET_GPIO_Port GPIOE
#define AD7606_CVA_START_Pin GPIO_PIN_13
#define AD7606_CVA_START_GPIO_Port GPIOE
#define AD7606_STBY_Pin GPIO_PIN_14
#define AD7606_STBY_GPIO_Port GPIOE
#define AD7606_RANGE_Pin GPIO_PIN_15
#define AD7606_RANGE_GPIO_Port GPIOE

//AD7606相关参数
#define CHANNEL_COUNT 4 //通道数
#define DUMMY_CONVERTIONS 4 // 初始冗余转换次数

#define AD7606_CH1 0
#define AD7606_CH2 1
#define AD7606_CH3 2
#define AD7606_CH4 3

static volatile uint16_t adc_value[CHANNEL_COUNT] = {0}; // 存储AD7606转换后的数据
float voltage[CHANNEL_COUNT] = {0.0}; // 存储电压值
volatile uint8_t adc_busy_irq_flag = 0;//AD7606 BUSY引脚中断标志位，触发外部中断后被置位

/**
 * @brief AD7606 硬件复位函数
 */
void AD7606_Reset(void)
{
    HAL_GPIO_WritePin(AD7606_RESET_GPIO_Port, AD7606_RESET_Pin, GPIO_PIN_RESET);
    HAL_Delay(1); // 等待1毫秒
    HAL_GPIO_WritePin(AD7606_RESET_GPIO_Port, AD7606_RESET_Pin, GPIO_PIN_SET);
    HAL_Delay(1);
    HAL_GPIO_WritePin(AD7606_RESET_GPIO_Port, AD7606_RESET_Pin, GPIO_PIN_RESET);
    // Usart_Printf(&huart1, "AD7606 Reset\r\n");
}

/**
 * @brief AD7606 初始化函数,设置采样范围为±5v，关闭过采样，关闭休眠模式
 */
void AD7606_Init(void)
{
    HAL_GPIO_WritePin(AD7606_STBY_GPIO_Port, AD7606_STBY_Pin, GPIO_PIN_SET); // 使能AD7606工作模式
    HAL_GPIO_WritePin(AD7606_CSN_GPIO_Port, AD7606_CSN_Pin, GPIO_PIN_SET);//CS引脚拉高

    //配置过采样引脚，不开启过采样
    HAL_GPIO_WritePin(AD7606_OS0_GPIO_Port, AD7606_OS0_Pin, GPIO_PIN_RESET);
    HAL_GPIO_WritePin(AD7606_OS1_GPIO_Port, AD7606_OS1_Pin, GPIO_PIN_RESET);
    HAL_GPIO_WritePin(AD7606_OS2_GPIO_Port, AD7606_OS2_Pin, GPIO_PIN_RESET);

    // 设置采样范围为±10v：高电平±10V，低电平±5V
    HAL_GPIO_WritePin(AD7606_RANGE_GPIO_Port, AD7606_RANGE_Pin, GPIO_PIN_RESET);
    // Usart_Printf(&huart1, "AD7606 Range set to ±5V\r\n");
    AD7606_Reset();
    HAL_Delay(10);
}

/**
 * @brief 启动AD7606转换
 */
void AD7606_StartConvert(void)
{
    // 拉低convst引脚，产生转换开始信号
    HAL_GPIO_WritePin(AD7606_CVA_START_GPIO_Port, AD7606_CVA_START_Pin, GPIO_PIN_RESET);
    HAL_GPIO_WritePin(AD7606_CVA_START_GPIO_Port, AD7606_CVA_START_Pin, GPIO_PIN_SET);
    //而后会触发AD7606的BUSY引脚中断，读取数据
}

/**
 * @brief 处理AD7606的BUSY引脚中断
 * @param GPIO_Pin 触发中断的引脚
 */
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    if (GPIO_Pin == AD7606_BUSY_Pin) // 判断是否是AD7606的BUSY引脚触发的中断
    {
        adc_busy_irq_flag=1;
    }
}

/**
 * @brief 获取AD7606转换后的数据
 * @return uint16_t* 指向存储ADC数据的数组
 */
const float* AD7606_GetData(void)
{
    // TinyCmd_Report("BUSY中断触发,启动SPI接收\r\n"); // 确认调试是否进入中断
    // 读取AD7606的数据
    HAL_GPIO_WritePin(AD7606_CSN_GPIO_Port, AD7606_CSN_Pin, GPIO_PIN_RESET);                                 // 片选拉低
    HAL_StatusTypeDef status = HAL_SPI_Receive(&hspi4, (uint8_t *)adc_value, CHANNEL_COUNT, 1000); // 读取数据
    HAL_GPIO_WritePin(AD7606_CSN_GPIO_Port, AD7606_CSN_Pin, GPIO_PIN_SET);                                   // 接收完成后释放片选

    if (status != HAL_OK)
    {
        TinyCmd_Report("SPI接收启动失败: %d\r\n", status);
    }
    else
    {
        // 将接收到的数据转换为电压值
        for (uint8_t i = 0; i < CHANNEL_COUNT; i++)
        {
            int16_t raw_adc_value = (int16_t)adc_value[i]; // 原始ADC值是16位有符号整数
            // 转换为电压值，假设参考电压为5V，分辨率为16位
            voltage[i] = (float)(raw_adc_value * 5.0f / 32768.0f);
        }
        return (const float *)voltage;
    }
    return NULL;
}
```

```c
/*main.c*/
int main(void)
{
	// 其他必要初始化，如SPI
	// 初始化ad7606
	AD7606_Init();

	/* Infinite loop */
	/* USER CODE BEGIN WHILE */
	while (1)
	{
		AD7606_StartConvert();//开启转换
		HAL_Delay(100);
		const float *ad7606_data = NULL;

		ad7606_data = AD7606_GetData();//获取数据

		if(NULL!=ad7606_data)
		{
			//处理数据
		}
		else
		{
			//错误处理
		}
	/* USER CODE END WHILE */
  }
}
```

### 问题分析

最开始的时候我使用的不是这个逻辑，而是用了SPI4的中断。
1.首先把start_convert函数放在main函数中循环，等待busy引脚触发。
2.然后在外部中断引脚的回调函数中，直接触发转换，拉低CS，使用HAL_SPI_Receive_IT()函数触发接收。这个函数和HAL_SPI_Receive()的区别是：该函数SPI传输是在后台进行的，传输完成后会触发SPI_Rx中断，通知程序传输完成，没有IT的版本是阻塞式传输的。
3.在SPI的传输完成中断回调函数中拉高片选，结束传输，并将一个adc_data_ready_flag置位。
4.main函数中轮询adc_data_ready_flag，被置位时处理数据。

使用这种思路遇到的问题是adc_data_ready_flag不会被置位，无法取到有用的数据。

最开始遇到这种问题，我首先考虑到是否是中断优先级的问题。
在我的项目中涉及中断如下：
1.USART1空闲中断（cmd，Debug接口）：0，0
2.SPI4接收中断（AD7606数据接收）：1，0
3.EXTI15_10外部中断（AD7606 Busy引脚中断）：1，0
我一直以为是串口和SPI的操作逻辑造成了死锁，最终尝试和很多种优先级都会卡在SPI回调，最终还是没有直接解决这个问题，于是我换了一个思路。

考虑到项目中另一个HDC1080的温湿度计，连中断都没开，也可以正常读取。于是我就放弃了SPI中断，转而在Busy引脚中置位标志量。剩下的操作放在main中。也就是现在的[[#基本思路]]。

现在想来，原本的最开始的想法其实很糟糕，有两个大的问题：
1.将一个器件的完整的通信时序拆分在两个中断（回调）函数中完成。这是很错误的操作，中断回调函数的触发完全不受控，而通信时序又对时机有很高的要求。因此，应该将对同一个器件的一次完整通信过程看作具有一定”原子性“的操作，尽量在同一代码块中完成。
2.在中断回调中做了太多操作。虽然在我看来这点操作量不会导致问题，但为了解决问题，我还是把所有操作都放在main中，只在回调中做了置位操作。

最终导致SPI回调卡住的直接原因还是没有找到，但侧面绕过了这个问题，就不必要非要深究那些莫名其妙的bug了。虽然”不要在中断函数中进行复杂操作“这句话都听烂了，但其实没怎么放在心上，果然有些坑只有自己踩过才长记性。