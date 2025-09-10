---
tags:
  - keil
  - stm32
title: keil新建stm32单片机工程
excerpt: 如何使用keil新建stm32单片机工程
categories: [嵌入式, 单片机, STM32]
date: 2025-09-10 21:56:46
---

# 新建stm32工程

## 工程分类

标准库和HAL库
1.使用标准库需要在ST官网下载相关库
[官方地址](https://www.st.com.cn/zh/embedded-software/stm32-standard-peripheral-libraries.html)

2.使用HAL库可以下载STM32cubeMX直接生成工程模板，需要在其中选择芯片型号，配置时钟源等基本配置。cubemx不止可以生成keil工程，还可以生成其他IDE的工程。

## 实际操作（标准库）

### 1.新建工程

选择合适的存放位置，路径上不要有中文（keil其实支持中文路径，但最好养成习惯），其他略

### 2.导入库函数

库函数类型的说明：

* 启动文件（startup）：`.s`结尾，以汇编语言编写，例如`startup_stm32f4xx.s`。根据芯片类型不同选择不同的文件，一个工程里只需要一个，是程序的开始的位置。

* 系统初始化文件（system_stm32f4xx.c）：系统时钟初始化配置文件。

* 内核寄存器配置文件（core_xx.h,core_xx.c）：根据不同的内核选择不同的文件，f10x系列单片机有`core_cm3.c,core_cm3.h`两个文件，而f4xx系列只有`core_cm4.h`

* 芯片头文件（stm32f4xx.h）：其中是很多宏定义，用来定义芯片寄存器、外设等信息。

* 库配置文件（stm32f4xx_conf.h 或 stm32f4xx_hal_conf.h）：各种头文件的包含关系，根据宏定义决定启用/禁用库相应的模块。

* 中断函数文件（stm32f4xx_it.h,stm32f4xx_it.c）：存放中断函数的文件

* 外设驱动文件(stm32f4xx_adc)：命名规则：芯片型号_外设 其中包含相应的外设的功能函数，使用这些函数，通过配置结构体即可控制相应的外设，而不必直接与寄存器打交道。根据具体需求添加，如 stm32f4xx_rcc.c、stm32f4xx_gpio.c。

* 主程序文件（main.c）：用户应用代码文件。

创建文件夹，将所需的文件放在相应的文件夹中，方便管理，这一步可以根据自己的习惯分类
主要分为三类:

* Start：系统初始化相关的文件，例如`core_cm3.c/.h`,`startup_stm32xxx.s`,`system_stm32fxx.c/.h`,

* Library:外设驱动函数和内核库函数，例如：`misc.c/h`,`stm32f4xx_rcc.c`,`stm32f4xx_gpio.c`

* User:用户新建或常用的函数，例如：`main.c`,`stm32f4xx.h`

> [!note]
keil不会同步当前工程增加文件的操作，需要主动添加文件
使用vscode创建工程, 成功包含了头文件但是右键函数等无法跳转可以尝试尝试重新打开工程

### 3.相关配置

1.有些宏需要配置到keil中，通过查看`stm32f4xx.h`(不同型号单片机该文件名不同，一般都是根据芯片类型选择)，可以看到其中的宏定义。

```c
#ifdef USE_STDPERIPH_DRIVER
  #include "stm32f4xx_conf.h"
#endif /* USE_STDPERIPH_DRIVER */
```

点击“魔术棒”按钮,选择"c/c++",在“define”栏中填入上述宏定义`USE_STDPERIPH_DRIVER`
不论型号，几乎所有的STM32项目该宏都是通用的，所以这一步可以看作固定步骤。

2.可以尝试编译，发现报错`Please select first the target STM32F4xx device used in your application (in stm32f4xx.h file)`
原因是在`stm32f4xx.h`中定义了f4xx系列单片机的各种寄存器，f4xx系列也有很多子系列单片机，其中不同型号的单片机的寄存器配置也不相同，因此需要一个宏来确认你所使用的单片机的具体型号型号，根据报错来确定位置，找到对应芯片的宏。

```c
#if !defined(STM32F40_41xxx) && !defined(STM32F427_437xx) && !defined(STM32F429_439xx) && !defined(STM32F401xx) && !defined(STM32F410xx) && \
    !defined(STM32F411xE) && !defined(STM32F412xG) && !defined(STM32F413_423xx) && !defined(STM32F446xx) && !defined(STM32F469_479xx)
 #error "Please select first the target STM32F4xx device used in your application (in stm32f4xx.h file)"
#endif /* STM32F40_41xxx && STM32F427_437xx && STM32F429_439xx && STM32F401xx && STM32F410xx && STM32F411xE && STM32F412xG && STM32F413_23xx && STM32F446xx && STM32F469_479xx */

```

例如，点击“魔术棒”按钮,选择"c/c++",在“define”栏中填写宏定义`STM32F429_439xx`，代表该型号被选定，其所拥有的特定寄存器可用。

3.使用微库`MicroLIB`
选择“魔术棒”按钮->target选项->Use MicroLIB

Target中选中微库“ Use MicroLib”，为的是在日后编写串口驱动的时候可以使用printf函数。而且有些应用中如果用了STM32的浮点运算单元FPU， 一定要同时开微库，不然有时会出现各种奇怪的现象。FPU的开关选项在微库配置选项下方的“Use Single Precision”中，默认是开的。

4.包含头文件
在C/C++选项卡中添加处理宏及编译器编译的时候查找的头文件路径。

![20240904103312](https://fuyunyou-note.oss-cn-wuhan-lr.aliyuncs.com/typora-user-images/20240904103312.png)

“Include Paths ”这里添加的是头文件的路径，如果编译的时候提示说找不到头文件，一般就是这里配置出了问题。你把头文件放到了哪个文件夹， 就把该文件夹添加到这里即可。(请使用图中的方法用文件浏览器去添加路径，不要直接手打路径，容易出错)

5.生成二进制文件
在Output选项卡中把输出文件夹定位到我们工程目录下的“output”文件夹， 如果想在编译的过程中生成hex文件，那么那Create HEX File选项勾上。
hex文件用于烧录
![20240904103514](https://fuyunyou-note.oss-cn-wuhan-lr.aliyuncs.com/typora-user-images/20240904103514.png)

### 4.编译验证

上述配置完成后尝试编译，看看会遇到什么报错，依次解决。

可能的报错

1. 宏重复定义
参考[网络文章](https://blog.csdn.net/jiaojunh/article/details/124207890),可能是官方库函数的bug，出现了同名的宏定义，于是产生冲突，把相应的宏注释掉即可

2. `stm32f4xx_fsmc.c`文件报错(f429型号报错)
参考网络文章：`https://bbs.eeworld.com.cn/thread-427197-1-1.html`
原因是不同型号的芯片功能不同，需要使用宏来区分，f429使用的fmc外设，没有fsmc外设，与其fsmc相关的宏定义会报错。
解决方法，右键单击该文件，选择不加入编译。或者可以直接将文件从工程中移除

3. `stm32f4xx_fmc.c`文件报错
原因同上，解决方法相同

STM32F429比较特殊，它有用FMC外设代替了FSMC外设的功能，所以它的库文件与其它型号的芯片不一样，在添加外设文件时， stm32f4xx_fmc.c和stm32f4xx_fsmc.c文件只能存在一个，而且我们的STM32F429芯片必须用fmc文件。如果我们把外设库的所有文件都添加进工程， 也可以使用下面的方法，设置文件不加入编译，这样也不会导致编译问题。这种设置在开发时也很常用，暂时不把文件加进编译，方便调试。

4. 链接错误

`..\Output\STM32F429_439xx-DEMO.axf: Error: L6218E: Undefined symbol TimingDelay_Decrement (referred from stm32f4xx_it.o).`
原因在于`TimingDelay_Decrement`函数出现异常

找到stm32f4xx_it.c 这个文件里的如下函数

  ```c
  void SysTick_Handler(void)
  {
    TimingDelay_Decrement（）；
  }
  ```

去删掉或者注释掉TimingDelay_Decrement（）；这个函数应该就可以了。
5. 链接错误

> [!error] 链接报错
> start linking ...
"d:/work/CO2/CO2_project/build/Debug/CO2_project.sct", line 7 (column 8): Error: L6235E: More than one section matches selector - cannot all be FIRST/LAST.
Not enough information to list image symbols.
Not enough information to list the image map.
Finished: 2 information, 0 warning and 1 error messages.

以上报错的大致含义是: 在工程的链接脚本`CO2_project.sct`中有不止一个`FIRST/LAST section`

可能的原因是包含了不止一个启动文件, 启动文件时以汇编语言编写的, 文件名形如`startup_stm32f429_439xx.s`，只在文件中保留一个与使用单片机型号相同的文件即可，其他从文件中移除。