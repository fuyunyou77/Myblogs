---
tags:
  - DAC
  - SPI
  - stm32
  - HAL库
title: STM32F429 HAL库硬件SPI操作AD5676记录
excerpt: hal库操作AD5676数模转换模块的软硬件编程记录
categories: [嵌入式, 单片机, STM32]
date: 2025-09-10 21:49:30
---
## 型号分类

AD5676是用于输出电压的DAC，8通道，16位输出精度，满量程输出为2.5V或5V。
还有另一个型号位AD5672，8通道，12位输出精度。
![image.png](https://fuyunyou-note.oss-cn-wuhan-lr.aliyuncs.com/typora-user-images/202508301518497.png)

## 硬件连线

![image.png](https://fuyunyou-note.oss-cn-wuhan-lr.aliyuncs.com/typora-user-images/202508301520540.png)

## 引脚介绍

![image.png](https://fuyunyou-note.oss-cn-wuhan-lr.aliyuncs.com/typora-user-images/202508301522444.png)

## 软件编程

### 配置硬件SPI

![image.png](https://fuyunyou-note.oss-cn-wuhan-lr.aliyuncs.com/typora-user-images/202508301524777.png)

### 拼接命令字

设置电压值需要
1.命令字：采用何种方式设置通道电流，可以控制DAC各种操作。一般选择0011："写入并更新DAC通道n"
![image.png](https://fuyunyou-note.oss-cn-wuhan-lr.aliyuncs.com/typora-user-images/202508301527286.png)

2.地址字：要设置的通道
![image.png](https://fuyunyou-note.oss-cn-wuhan-lr.aliyuncs.com/typora-user-images/202508301528018.png)
数据手册中说，可以使用地址位来选择任意组合的DAC通道。感觉这句话有点问题，因为把通道组合起来能表示的信息不唯一。
例如：传入0011地址，究竟表示单独操作通道3，还是通道1（0001）和通道2（0002）两者一起操作？
因此编程中还是一次操作一个通道。

3.数据字：要设置的电压值，需要将浮点数转化为uint16_t

```c
static uint16_t AD5676_Vol2Code(float vol)
{
    uint16_t code = 0;
    code = (uint16_t)(((vol * 65535.0f) / 5.0f));
    return code;
}
```

![image.png](https://fuyunyou-note.oss-cn-wuhan-lr.aliyuncs.com/typora-user-images/202508301538077.png)
最终要发送的数据刚好24字节，使用3字节大小的数组存储并按顺序发送即可。

### 软件代码

```c
#define AD_5676_CH1_ADDR 0x0
#define AD_5676_CH2_ADDR 0x1
#define AD_5676_CH3_ADDR 0x2
#define AD_5676_CH4_ADDR 0x3
#define AD_5676_CH5_ADDR 0x4
#define AD_5676_CH6_ADDR 0x5
#define AD_5676_CH7_ADDR 0x6
#define AD_5676_CH8_ADDR 0x7

// 定义AD5676的命令
#define AD_5676_CMD_W_INPUT_REG 0x01 // 写入输入寄存器
#define AD_5676_CMD_U_DAC_REG 0x02 // 以输入寄存器n的值更新DACn的输出
#define AD_5676_CMD_W_U_SINGLE 0x03 // 写入并直接更新通道输出
#define AD_5676_CMD_W_U_ALL_INPUT_REG 0x10 // 用输入数据写入并更新所有通道的输入寄存器
#define AD_5676_CMD_W_U_ALL_INPUT_DAC_REG 0x11 // 用输入数据写入并更新所有通道的输入寄存器和DAC寄存器
#define AD_5676_SOFT_RESET 0x06 // 软复位

// 定义AD5676的SPI数据帧拼接宏
#define AD_5676_ADDR(x) ((x) & 0x0F)
#define AD_5676_CMD(x) (((x) << 4) & 0xF0)
#define AD_5676_VAL_H(x) (((x) >> 8) & 0xFF)
#define AD_5676_VAL_L(x) ((x) & 0xFF)

void AD5676_Reset(void)
{
    // 复位
    HAL_GPIO_WritePin(AD5676_RSTN_GPIO_Port, AD5676_RSTN_Pin, GPIO_PIN_RESET);
    HAL_Delay(1); // TODO：引入RTOS后delay全部替换
    HAL_GPIO_WritePin(AD5676_RSTN_GPIO_Port, AD5676_RSTN_Pin, GPIO_PIN_SET);
    HAL_Delay(1);
}

void AD5676_Init(void)
{
    // 初始化AD5676的GPIO引脚
    HAL_GPIO_WritePin(AD5676_CS_GPIO_Port, AD5676_CS_Pin, GPIO_PIN_SET); // 片选引脚拉高
    HAL_GPIO_WritePin(AD5676_SCLK_GPIO_Port, AD5676_SCLK_Pin, GPIO_PIN_RESET); // 时钟引脚拉低
    HAL_GPIO_WritePin(AD5676_SDI_GPIO_Port, AD5676_SDI_Pin, GPIO_PIN_RESET); // 数据输入引脚拉低

    // 复位AD5676
    AD5676_Reset();
}

static uint16_t AD5676_Vol2Code(float vol)
{
    uint16_t code = 0;
    code = (uint16_t)(((vol * 65535.0f) / 5.0f));
    return code;
}


HAL_StatusTypeDef AD5676_SetOutput(uint8_t channel_addr, float vol)
{
    HAL_StatusTypeDef result = HAL_ERROR;
    uint8_t buf[3];
    uint16_t code = AD5676_Vol2Code(vol);

    buf[0]=AD_5676_CMD(AD_5676_CMD_W_U_SINGLE) | AD_5676_ADDR(channel_addr);
    buf[1]=AD_5676_VAL_H(code);
    buf[2]=AD_5676_VAL_L(code);

    HAL_GPIO_WritePin(AD5676_CS_GPIO_Port, AD5676_CS_Pin, GPIO_PIN_RESET);//拉低片选
    result = HAL_SPI_Transmit(&hspi2, buf, sizeof(buf), 100);
    HAL_GPIO_WritePin(AD5676_CS_GPIO_Port, AD5676_CS_Pin, GPIO_PIN_SET);//拉高片选

    return result;
}
```

最终输出的电压精度挺高的，用精度较高的万用表测，误差在0.1mv