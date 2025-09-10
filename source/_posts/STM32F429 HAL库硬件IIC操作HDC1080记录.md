---
tags:
  - IIC
  - stm32
  - HAL库
title: STM32F429 HAL库硬件IIC操作HDC1080记录
excerpt: hal库操作HDC1080温湿度计的软硬件编程记录
categories: [嵌入式, 单片机, STM32]
date: 2025-09-10 21:48:10
---
## 器件类型

HDC1080是14bit采样分辨率的温湿度传感器，通过IIC通信
![image.png](https://fuyunyou-note.oss-cn-wuhan-lr.aliyuncs.com/typora-user-images/202508301618884.png)

## 硬件连线

![image.png](https://fuyunyou-note.oss-cn-wuhan-lr.aliyuncs.com/typora-user-images/202508301619207.png)

## 引脚介绍

HDC1080的引脚非常简单，只需要控制IIC的时钟和数据就好了。
![image.png](https://fuyunyou-note.oss-cn-wuhan-lr.aliyuncs.com/typora-user-images/202508301621985.png)

## 软件编程

HDC1080中有如下寄存器
![image.png](https://fuyunyou-note.oss-cn-wuhan-lr.aliyuncs.com/typora-user-images/202508301625320.png)

其中config寄存器，用来进行各种读取操作的配置，如下所示：
![image.png](https://fuyunyou-note.oss-cn-wuhan-lr.aliyuncs.com/typora-user-images/202508301633693.png)
要同时读取温度和湿度，需要配置寄存器第12位为1要设置读取分辨率为14位，需要设置\[10:8]都为0。
因此config寄存器中的值应该为0x1000，不过不用额外设置，该寄存器默认就是该值。

如果没有其他要求，HDC1080使用默认配置就很好，接下来直接配置硬件IIC，也保持默认配置即可。
![image.png](https://fuyunyou-note.oss-cn-wuhan-lr.aliyuncs.com/typora-user-images/202508301651860.png)

对读取到的数据进行转化：
![image.png](https://fuyunyou-note.oss-cn-wuhan-lr.aliyuncs.com/typora-user-images/202508301655085.png)

![image.png](https://fuyunyou-note.oss-cn-wuhan-lr.aliyuncs.com/typora-user-images/202508301655685.png)

### 软件代码

```c

#define HDC1080_ADDR_R 0x81 // 1000 0001 高七位代表器件地址，最低为为读命令
#define HDC1080_ADDR_W 0x80 // 1000 0000 写命令

#define TEMPER_REG_ADDR 0x00
#define HUMIDITY_REG_ADDR 0x01
#define CONFIG_REG_ADDR 0x02
#define MENUFACTURE_ID_REG_ADDR 0xfe
#define DEVICE_ID_REG_ADDR 0xff

#define CONFIG_REG_VALUE 0x1000 // 配置寄存器设为0x1000，表示获取温度和湿度，精度都为14bit，发送时只发送高八位，低八位固定为0

typedef struct Temp_Humi_Value_t
{
    float temperature; // 温度值
    float humidity;    // 湿度值
} TH_Value_t;

/**
 * @brief 获取HDC1080的设备ID
 * @return uint16_t 设备ID
 * @retval 0x1050
 */
uint16_t HDC1080_Get_DeviceID()
{
    uint8_t buf[2];
    HAL_I2C_Mem_Read(&hi2c1, HDC1080_ADDR_R, DEVICE_ID_REG_ADDR, I2C_MEMADD_SIZE_8BIT, buf, sizeof(buf), 1000);
    return (buf[0] << 8) | buf[1];
}

/**
 * @brief 设置HDC1080的配置寄存器
 * @param Reg_value 配置寄存器值
 * @retval None
 * @note 配置寄存器值为0x1000，表示获取温度和湿度，精度都为14bit，发送时只发送高八位，低八位固定为0
*/
void HDC1080_Set_ConfigReg(uint16_t Reg_value)
{
    uint8_t buf[2];
    /*寄存器值存储在高八位，低八位固定为0,但根据时序要求,还是要把低八位也发送出去*/
    buf[0] = (Reg_value >> 8) & 0xFF;
    buf[1] = Reg_value & 0xFF;
    HAL_I2C_Mem_Write(&hi2c1, HDC1080_ADDR_W, CONFIG_REG_ADDR, I2C_MEMADD_SIZE_8BIT, buf, sizeof(buf), 1000);
}

/**
 * @brief 获取HDC1080的温度和湿度值
 * @param temper 温度值指针
 * @param humi 湿度值指针
 * @retval None
 * @note 温度和湿度值的单位分别为摄氏度和百分比
 */
TH_Value_t HDC1080_Get_THvalue(void)
{
    uint8_t buf[4];
    uint8_t reg_addr = TEMPER_REG_ADDR;
    TH_Value_t th_value;

    // 第一步：发送要读取的寄存器地址（温度寄存器）
    // 这会触发HDC1080开始进行温湿度转换
    HAL_I2C_Master_Transmit(&hi2c1, HDC1080_ADDR_W, &reg_addr, 1, 1000);

    // 等待转换完成，参考软件I2C的延迟时间
    // HDC1080在最高分辨率下转换时间约为15ms
    HAL_Delay(15);

    // 第二步：读取转换后的4字节数据（温度2字节 + 湿度2字节）
    HAL_I2C_Master_Receive(&hi2c1, HDC1080_ADDR_R, buf, sizeof(buf), 1000);

    // 温度计算
    uint16_t temp_raw = (buf[0] << 8) | buf[1];
    th_value.temperature = (float)((temp_raw * 165.0 / 65536.0)- 40.0); // 转换为摄氏度

    // 湿度计算
    uint16_t humi_raw = (buf[2] << 8) | buf[3];
    th_value.humidity = (float)(humi_raw * 100.0 / 65536.0);

    return th_value; // 返回温度和湿度值
}
```

## 注意事项

HDC1080对湿度进行转换需要一定的时间，在发出读命令后不要立即接收，需要等待较长的一段时间，此处延迟15ms。

Hal库中有一个HAL_I2C_Mem_Read()函数
函数原型：

```c
HAL_StatusTypeDef HAL_I2C_Mem_Read(I2C_HandleTypeDef *hi2c, uint16_t DevAddress, uint16_t MemAddress, uint16_t MemAddSize, uint8_t *pData, uint16_t Size, uint32_t Timeout);
```

该函数作用是直接读取对应器件中的寄存器值，其中包含了IIC读取寄存器的一次收发的完整时序。
但该函数在操作HDC1080时不可用，因为其中没有足够的延时，不能读取到正确的值。
