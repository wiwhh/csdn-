> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/weixin_46253745/article/details/135321134)

#### 文章目录

*   [1. STM32 程序升级方法](#1_STM32_4)
*   *   [1.1 ST-Link / J-link 下载](#11_STLink__Jlink_5)
    *   [1.2 ISP（In System Programing）](#12_ISPIn_System_Programing_18)
    *   [1.3 IAP（In Applicating Programing）](#13_IAPIn_Applicating_Programing_52)
    *   *   [1.3.1 正常程序运行流程](#131__58)
        *   [1.3.2 有 IAP 时程序运行流程](#132_IAP_69)
*   [2. STM32 Bootloader 实现](#2_STM32_Bootloader_97)
*   *   [2.1 方式一：Boot_App（已实现）](#21_Boot_App_112)
    *   *   [2.1.1 Bootloader](#211_Bootloader_117)
        *   [2.1.2 APP](#212_APP_551)
        *   [2.1.3 测试](#213__582)
    *   [2.2 方式二：其他接口 / USB 拖拽等（未完成）](#22___USB_597)

[**====>>> 文章汇总 <<<====**](https://blog.csdn.net/weixin_46253745/article/details/127891788)

1. STM32 程序升级方法
---------------

### 1.1 [ST-Link](https://so.csdn.net/so/search?q=ST-Link&spm=1001.2101.3001.7020) / J-link 下载

这个应该是最基本的方法，只要自己写过程序的应该都会，将编译生成的`hex`文件使用 **ST-Link** 工具或者 **J-Link** 工具直接下载进 Flash 即可。Keil 中点击下载也能一键下载。

下载时可以看到地址是从 **0x0800 0000**，即 Flash 的起始地址开始下载的。

**优点**：简单，插上下载器直接下载即可。  
**缺点**：在产品中嵌入式板卡封装起来之后，因为下载口没有实际功能，所以很多时候下不拆机是没办法插上下载器的。这时候就不方便。

简单补充一句，**bin 文件**和 **hex 文件**的区别：

*   bin 文件不带地址信息，因此下载的时候需要指定下载地址。
*   hex 文件自带地址信息，直接点击下载自己会找到要下载到的地址（默认 0x0800 0000）。

### 1.2 ISP（In System Programing）

我们常见的**一键下载电路**就是用的这种方式。这个是利用了 STM32 自带的 Bootloader 升级程序。

在用户参考手册中，可以看到下表，关于启动模式设置的。  
![](https://img-blog.csdnimg.cn/direct/d0053b51541645ec8a3e490b84fb1d42.png)  
中文版的如下：  
![](https://img-blog.csdnimg.cn/direct/a2378a9bba5d48a28eb8ec468839e446.png)  
**程序运行时的启动过程**：

一般在我们的程序运行的时候，BOOT0 是接地的，也就是`BOOT0 = 0`。也就是程序是从**主存储器 Flash** 开始启动的，启动的地址为`0x08000000`。

**使用 ISP 下载时的启动过程**：

1.  首先，硬件上将 STM32 的 BOOT0 引脚拉高、BOOT1 引脚拉低，即`BOOT0 = 1`、`BOOT1 = 0`。
2.  此时，程序会从系统存储器中的程序启动，这段程序会接收串口数据（我们编译好的程序文件），并将这写数据放到主闪存存储器（Flash）当中。
3.  最后，硬件上重新将 BOOT0 接地，也就是`BOOT0 = 0`，然后复位引脚拉低，程序复位重启，从 Flash 中开始运行程序。

> 可以这么理解：芯片出厂时，系统存储器中已经存储了一段程序，这段程序的功能是将**串口 1**（固定的）收到的数据，放到主闪存存储器（Flash）中，从 0x0800 0000 地址处开始。

这样也就完成了我们的一键下载过程。  
比如，正点原子的一键下载电路如下，左侧的三极管就是用来完成 BOOT 引脚和复位引脚的操作的。

![](https://img-blog.csdnimg.cn/direct/d6d3aaac67bc4e8b86cc72fd566b0b11.jpeg)

> 具体的硬件引脚电平如何变化百度搜一下，分析应该挺多的。我也没详细看过。

沁恒官网也有专门的一键下载芯片，原理上其实一样的。  
![](https://img-blog.csdnimg.cn/direct/9b52ef6cefc54b4991490d077bc940b0.jpeg)

> 链接：https://www.wch.cn/application/575.html

ISP 的下载方式：

**优点**：提供了一种升级方式，无需代码支持。  
**缺点**：需要相应的硬件支持，成本增加；使用的接口也是固定的，并且很多时候串口可能用于其他功能了已经。

### 1.3 IAP（In Applicating Programing）

IAP 和 ISP 其实基本上是一样的。

但是：ISP 是由厂商已经提供好的，因此接口固定；IAP 可以自定义使用任何接口接收应用程序。也正是因为这一点，使得用户可以用多种不同的方式进行升级。

#### 1.3.1 正常程序运行流程

在看 IAP 之前，要先看一下正常情况下，程序从 Flash 启动时的启动流程，如下图所示：

1.  首先程序从 Flash 启动，根据**中断向量表**找到**复位中断处理函数**的地址（0x0800 0004 处是中断向量表的起始地址，记录了复位中断处理函数的地址）。
2.  执行复位中断处理函数，初始化系统环境之后，该函数最后会跳转到 **main 函数**继续运行。
3.  在 main 函数的死循环中一直运行，直到**有中断发生**时（外设中断等等），重新跳转到**中断向量表起始**处。
4.  在中断向量表中根据**中断信号源**来判断要执行的中断处理函数，然后跳转到**相应的中断处理函数**。
5.  中断处理函数运行完成之后继续跳转到 **main 函数**处继续运行。

![](https://img-blog.csdnimg.cn/direct/4169eae194cc44269651d9976a76f0f9.png)

#### 1.3.2 有 IAP 时程序运行流程

接下来看下有了 IAP 之后的启动流程，如下图所示，下图中可以看到，在 Flash 中存储了 两 “套” 程序（套的意思是，不仅只有用户程序，配套的中断向量等也都有）。

其中第一套为：Bootloader 程序，该程序的功能是，接收某个接口的数据，并把这些数据存储到 Flash 中。

> 这些数据其实就是后面的**一套** APP 程序，这套应用程序可以通过 Bootlaoder 程序保存到 Flash 中，这样就不用再使用专门的下载器。

第二套才是我们真正的应用程序。

1.  首先程序从 Flash 启动，根据**中断向量表**找到**复位中断处理函数**的地址（0x0800 0004 处是中断向量表的起始地址，记录了复位中断处理函数的地址）。
2.  执行复位中断处理函数，该函数最后会跳转到 **Bootloader 的 main 函数**继续运行。这个 main 函数的任务就是，判断是否接收新的 APP 程序，如果有，就把新的 APP 程序文件保存到 Flash（就是第二套程序的位置）然后跳转到第二套程序中运行。如果无，就直接跳转到第二套程序中运行。
3.  跳转之后的过程也是和正常程序运行的流程一样，一旦进入新的 APP 程序的起始地址，就会根据中断向量表找到复位中断处理函数，然后进入 **App 程序的 main 函数**运行。
4.  但是如果发生中断，是强制跳转到 **Bootloader 程序的中断向量表**进行查询的，而我们需要的肯定是需要跳转到 **APP 程序的中断处理函数**处运行。**所以，在进行到 APP 程序后，APP 程序一定要修改中断向量表的偏移，让查找对应中断处理函数的时候偏移一段地址到 App 程序的中断处理函数处，否则 APP 程序中的中断发生时，就无法跳转到 APP 程序的中断处理函数了。**

![](https://img-blog.csdnimg.cn/direct/172388cfc5c444ccb75bcde6d04e7de9.png)  
到这，我们就知道 Bootloader 程序是干啥的了。

> 这个分析过程，是 2.1 小节中的方式一。其他方式跟这种原理上基本是一样一样的。

对比：

<table><thead><tr><th>ISP</th><th>IAP</th></tr></thead><tbody><tr><td>在系统存储器中存储了一套接收串口 1 数据的程序</td><td>IAP 是将 Flash 分成了两份，在第一份中存储了一套接收某个接口的程序</td></tr><tr><td>使用硬件 BOOT 引脚设置进行跳转</td><td>使用软件（直接修改 PC 指针）进行跳转</td></tr><tr><td>仅能使用芯片厂商设置好的接口（串口 1）</td><td>用户自定义，理论上只要能接收数据的接口都可以用</td></tr></tbody></table>

2. STM32 Bootloader 实现
----------------------

开发板：STM32F401CCU6 最小系统板（淘宝十几块钱），Flash 大小：256KB，RAM 大小：64KB。

TODO：这里记录一下，我看到的或者想到的所有形式。每实现一个会过来贴代码地址。

1.  Bootloader_App
2.  Bootloader_Setting_App
3.  Bootloader_App1_App2
4.  U 盘拖拽
5.  无线升级
6.  …

**代码汇总地址：https://gitee.com/HzoZi/bootloader**

### 2.1 方式一：Boot_App（已实现）

这应该是最常见、也是最简单的一种了。也就是上面 1.3.2 小节的分析的情况。

![](https://img-blog.csdnimg.cn/direct/dbf3f5534b32450ab6bb012dbbca6870.png)

#### 2.1.1 Bootloader

需要实现四个部分：

1.  串口接收程序，用于接收 App 程序；
2.  STM32 Flash 写入接口，用于将 App 程序写入到 Flash；
3.  App 跳转实现，用于跳转到 App 程序处开始运行。
4.  设置 Keil 参数。

**第一步：新建工程**

调试口勾上，时钟设置最大，设置生成单独的. c 文件。

1.  勾选一个串口，参数默认即可。  
    ![](https://img-blog.csdnimg.cn/direct/cd205161f1cc46f491e4089833ecdb87.png)
2.  串口中断也勾上

![](https://img-blog.csdnimg.cn/direct/6d352a2180ef451e90cbef32d283f134.png)  
3. 设置一个按键

![](https://img-blog.csdnimg.cn/direct/c10408071a2545b7b65fc9453d329768.png)  
生成工程即可。

**第二步：配置串口，实现串口接收功能**

1.  添加 printf 函数支持

我这里放到了 usart.c 的最后。  
![](https://img-blog.csdnimg.cn/direct/825100e31acb448989cc434890b51cb2.png)

```
// 需要调用stdio.h文件
#include <stdio.h>
// 取消ARM的半主机工作模式
#pragma import(__use_no_semihosting)
struct __FILE
{
	int handle;
};
FILE __stdout;
void _sys_exit(int x) // 定义_sys_exit()以避免使用半主机模式
{
	x = x;
}

int fputc(int ch, FILE *f)
{
	HAL_UART_Transmit(&huart1, (uint8_t *)&ch, 1, 0xffff);
	return ch;
}

```

2.  实现中断接收 App 程序

我这里放到了 main.c 的上面。  
![](https://img-blog.csdnimg.cn/direct/b1c2ec818b0e47aea304640f82c5453d.png)

```
uint8_t single_buff[1];					// 按字节保存APP程序
uint8_t app_buff[40 * 1024] = {0};		// 保存接收到的APP程序（最大40K）
volatile uint32_t app_buff_len = 0;		// APP程序的长度（字节）

void start_uart_rx(void)
{
	while(HAL_UART_Receive_IT(&huart1, single_buff, sizeof(single_buff)) != HAL_OK);
}

void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
	if(huart->Instance == USART1)
	{
		// 把接收到的数据，放到缓冲区中
		app_buff[app_buff_len] = single_buff[0];
		app_buff_len++;
	}
	
	// 重新开启中断接收
	start_uart_rx();
}

```

**第三步：IAP 实现**

其中，STM32Flash 读写部分 其实有现成的代码可以用。使用 RT_Thread Studio 创建一个一样芯片的工程就可以直接抄了。

![](https://img-blog.csdnimg.cn/direct/704db1c475c048b69ca326c5db983fa4.png)  
我们新建一个文件，存放 IAP 实现的相关代码。

1.  头文件：

**注意，这里划分 Flash 的时候，尤其注意 App 程序起始地址，根据芯片不同起始地址倍数关系也不同**

*   STM32F1：地址必须是 4 的倍数，因为每次写入只能写入 32 位数据，即 4 个字节。
*   STM32F4：地址可以从任意地址开始，因为每次写入可以写入 8 位数据，每个地址就是 8 位数据。
*   STM32L4：地址必须是 8 的倍数，因为每次写入只能写入 64 位数据，即 8 个字节。

![](https://img-blog.csdnimg.cn/direct/69db5371b31a4faaa0d9883adeb7b39b.png)

```
#ifndef __IAP_H
#define __IAP_H

#include "stm32f4xx_hal.h"
#include "stdint.h"

/* 以下宏定义名字尽量不要随便换 */

/* Flash定义，根据使用的芯片修改 */
#define ROM_START              			((uint32_t)0x08000000)
#define ROM_SIZE               			(256 * 1024)
#define ROM_END                			((uint32_t)(ROM_START + ROM_SIZE))
#define STM32_FLASH_START_ADRESS		ROM_START
#define STM32_FLASH_SIZE				ROM_SIZE
#define STM32_FLASH_END_ADDRESS			ROM_END


/* Flash扇区定义，STM32F401CCU6：256KB，根据使用的芯片修改 */
#define ADDR_FLASH_SECTOR_0     ((uint32_t)0x08000000) /* Base @ of Sector 0, 16 Kbytes */
#define ADDR_FLASH_SECTOR_1     ((uint32_t)0x08004000) /* Base @ of Sector 1, 16 Kbytes */
#define ADDR_FLASH_SECTOR_2     ((uint32_t)0x08008000) /* Base @ of Sector 2, 16 Kbytes */
#define ADDR_FLASH_SECTOR_3     ((uint32_t)0x0800C000) /* Base @ of Sector 3, 16 Kbytes */
#define ADDR_FLASH_SECTOR_4     ((uint32_t)0x08010000) /* Base @ of Sector 4, 64 Kbytes */
#define ADDR_FLASH_SECTOR_5     ((uint32_t)0x08020000) /* Base @ of Sector 5, 128 Kbytes */


/* Bootloader、APP 分区定义，根据个人需求修改 */
#define BOOT_START_ADDR			0x08000000		// FLASH_START_ADDR
#define BOOT_FLASH_SIZE			0x4000			// 16K
#define APP_START_ADDR			0x08004000		// BOOT_START_ADDR + BOOT_FLASH_SIZE
#define APP_FLASH_SIZE			0x3C000			// 240K


/* 对外接口 */
void show_boot_info(void);
uint8_t jump_app(uint32_t app_addr);

/* 对外接口 */
int stm32_flash_read(uint32_t addr, uint8_t *buf, size_t size);
int stm32_flash_write(uint32_t addr, const uint8_t *buf, size_t size);
int stm32_flash_erase(uint32_t addr, size_t size);

#endif /* __IAP_H */

```

2.  源文件：

我这里直接从 RT-Thread 的程序中抄的，要注意，不同系列芯片的读写函数略有差异，主要就是地址差异，App 起始地址只要没问题，这里可以不用考虑，知道这回事就行。

![](https://img-blog.csdnimg.cn/direct/a8a44bfdb4de45baa8355cb0bb691340.png)

抄过来之后如下图所示

![](https://img-blog.csdnimg.cn/direct/3b7fb943a1b04059947aa6b340fe8368.png)

![](https://img-blog.csdnimg.cn/direct/dae04d253fbd478aafac6f5c81565209.png)

```
#include "iap.h"
#include "stdio.h"

void show_boot_info(void)
{
    printf("---------- Enter BootLoader ----------\r\n");
    printf("\r\n");
    printf("======== flash pration table =========\r\n");
    printf("| name     | offset     | size       |\r\n");
    printf("--------------------------------------\r\n");
    printf("| boot     | 0x%08X | 0x%08X |\r\n", BOOT_START_ADDR, BOOT_FLASH_SIZE);
    printf("| app      | 0x%08X | 0x%08X |\r\n", APP_START_ADDR, APP_FLASH_SIZE);
    printf("======================================\r\n");
}

typedef void (*jump_callback)(void);
/**
 * @note 跳转至App运行
 *
 * @param App起始地址
 *
 * @return result
 */
uint8_t jump_app(uint32_t app_addr)
{
    uint32_t jump_addr;
    jump_callback cb;
	
    if (((*(volatile uint32_t *)app_addr) & 0x2FFE0000 ) == 0x20000000) 
	{
		// 复位向量位于程序起始地址+4处
        jump_addr = *(volatile uint32_t*)(app_addr + 4);
		
        cb = (jump_callback)jump_addr;
		
		// 设置主堆栈指针指向 APP 程序起始地址
        __set_MSP(*(volatile uint32_t*)app_addr);  
        
		cb();
		
        return 1;
    }
    return 0;
}

/**
  * @brief  Gets the sector of a given address
  * @param  None
  * @retval The sector of a given address
  */
static uint8_t GetSector(uint32_t Address)
{
    if((Address < ADDR_FLASH_SECTOR_1) && (Address >= ADDR_FLASH_SECTOR_0))
    {
        return FLASH_SECTOR_0;
    }
    else if((Address < ADDR_FLASH_SECTOR_2) && (Address >= ADDR_FLASH_SECTOR_1))
    {
        return FLASH_SECTOR_1;
    }
    else if((Address < ADDR_FLASH_SECTOR_3) && (Address >= ADDR_FLASH_SECTOR_2))
    {
        return FLASH_SECTOR_2;
    }
    else if((Address < ADDR_FLASH_SECTOR_4) && (Address >= ADDR_FLASH_SECTOR_3))
    {
        return FLASH_SECTOR_3;
    }
    else if((Address < ADDR_FLASH_SECTOR_5) && (Address >= ADDR_FLASH_SECTOR_4))
    {
        return FLASH_SECTOR_4;
    }
	else
	{
		return FLASH_SECTOR_5;
	}
}

/**
 * Read data from flash.
 * @note This operation's units is word.
 *
 * @param addr flash address
 * @param buf buffer to store read data
 * @param size read bytes size
 *
 * @return result
 */
int stm32_flash_read(uint32_t addr, uint8_t *buf, size_t size)
{
    size_t i;

    if ((addr + size) > STM32_FLASH_END_ADDRESS)
    {
        printf("read outrange flash size! addr is (0x%p)", (void*)(addr + size));
        return -1;
    }

    for (i = 0; i < size; i++, buf++, addr++)
    {
        *buf = *(uint8_t *) addr;
    }

    return size;
}

/**
 * Write data to flash.
 * @note This operation's units is word.
 * @note This operation must after erase. @see flash_erase.
 *
 * @param addr flash address
 * @param buf the write data buffer
 * @param size write bytes size
 *
 * @return result
 */
int stm32_flash_write(uint32_t addr, const uint8_t *buf, size_t size)
{
    int8_t result = 0;
    uint32_t end_addr = addr + size;

    if ((end_addr) > STM32_FLASH_END_ADDRESS)
    {
        printf("write outrange flash size! addr is (0x%p)", (void*)(addr + size));
        return -1;
    }

    if (size < 1)
    {
        return -1;
    }

    HAL_FLASH_Unlock();

    __HAL_FLASH_CLEAR_FLAG(FLASH_FLAG_EOP | FLASH_FLAG_OPERR | FLASH_FLAG_WRPERR | FLASH_FLAG_PGAERR | FLASH_FLAG_PGPERR | FLASH_FLAG_PGSERR);

    for (size_t i = 0; i < size; i++, addr++, buf++)
    {
        /* write data to flash */
        if (HAL_FLASH_Program(FLASH_TYPEPROGRAM_BYTE, addr, (uint64_t)(*buf)) == HAL_OK)
        {
            if (*(uint8_t *)addr != *buf)
            {
                result = -1;
                break;
            }
        }
        else
        {
            result = -1;
            break;
        }
    }

    HAL_FLASH_Lock();

    if (result != 0)
    {
        return result;
    }

    return size;
}

/**
 * Erase data on flash.
 * @note This operation is irreversible.
 * @note This operation's units is different which on many chips.
 *
 * @param addr flash address
 * @param size erase bytes size
 *
 * @return result
 */
int stm32_flash_erase(uint32_t addr, size_t size)
{
    int8_t result = 0;
    uint32_t FirstSector = 0, NbOfSectors = 0;
    uint32_t SECTORError = 0;

    if ((addr + size) > STM32_FLASH_END_ADDRESS)
    {
        printf("ERROR: erase outrange flash size! addr is (0x%p)\n", (void*)(addr + size));
        return -1;
    }

    /*Variable used for Erase procedure*/
    FLASH_EraseInitTypeDef EraseInitStruct;

    /* Unlock the Flash to enable the flash control register access */
    HAL_FLASH_Unlock();

    __HAL_FLASH_CLEAR_FLAG(FLASH_FLAG_EOP | FLASH_FLAG_OPERR | FLASH_FLAG_WRPERR | FLASH_FLAG_PGAERR | FLASH_FLAG_PGPERR | FLASH_FLAG_PGSERR);

    /* Get the 1st sector to erase */
    FirstSector = GetSector(addr);
    /* Get the number of sector to erase from 1st sector*/
    NbOfSectors = GetSector(addr + size - 1) - FirstSector + 1;
    /* Fill EraseInit structure*/
    EraseInitStruct.TypeErase     = FLASH_TYPEERASE_SECTORS;
    EraseInitStruct.VoltageRange  = FLASH_VOLTAGE_RANGE_3;
    EraseInitStruct.Sector        = FirstSector;
    EraseInitStruct.NbSectors     = NbOfSectors;

    if (HAL_FLASHEx_Erase(&EraseInitStruct, (uint32_t *)&SECTORError) != HAL_OK)
    {
        result = -1;
        goto __exit;
    }

__exit:
    HAL_FLASH_Lock();

    if (result != 0)
    {
        return result;
    }

    printf("erase done: addr (0x%p), size %d", (void*)addr, size);
    return size;
}


```

3.  main 函数：

![](https://img-blog.csdnimg.cn/direct/7f685e96c7be44218c6959b8c9459d0c.png)

```
#include "iap.h"
#include "stdio.h"

int main(void)
{
	// 这里只是我自己写的部分，cubemx自动生成的这里没有，记得填上
	start_uart_rx();	// 开始中断接收
	show_boot_info();	// 输出分区信息
  while (1)
  {
		printf("waitting input... \r\n");
	  
		// 判断是否需要更新程序（5s）
		for(uint16_t i = 0; i < 5000; i++)
		{
			// 如果按键按下, 说明需要更新程序
			if(0 == HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0))
			{
				HAL_Delay(20);
				if(0 == HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0))
				{
					// 按键按下了，更新程序并跳转（先传程序，再按下按键）
					// 擦除APP区域
					printf("erase app... \r\n");
					stm32_flash_erase(APP_START_ADDR, APP_FLASH_SIZE);
					HAL_Delay(100);
					
					// 写入APP程序
					printf("write app... \r\n");
					stm32_flash_write(APP_START_ADDR, app_buff, app_buff_len);//更新FLASH代码
					break;
				}
			}
			HAL_Delay(1);
		}
		
		// 跳转到APP
		if(0 == jump_app(APP_START_ADDR))
		{
			printf("jump app failed... \r\n");
			while(1);
		}
	}
}

```

4.  keil 配置：

这里 Bootloader 程序给分配大小是 16KB（看 iap 头文件），不是默认的全部 Flash，因此在 Keil 中修改一下。

![](https://img-blog.csdnimg.cn/direct/8c1e67f80e9f42aaa6fbec3beda5e237.png)  
至此，Bootloader 程序就完成了。

#### 2.1.2 APP

App 程序比较简单，创建一个 LED 闪烁工程 或 串口输出工程即可。

1.  修改中断向量表偏移

![](https://img-blog.csdnimg.cn/direct/688b463d0ac44484877b5330f8c70d79.png)

```
// 设置中断向量偏移量
#define NVIC_VTOR_MASK       0x3FFFFF80
#define APP_PART_ADDR        0x08004000
SCB->VTOR = APP_PART_ADDR & NVIC_VTOR_MASK;

```

2.  修改 Flash 起始地址和 Flash 大小

![](https://img-blog.csdnimg.cn/direct/34dad890888442a9bcbe8cd4c297bbcc.png)

3.  设置编译输出 bin 文件（App 只能用 bin 文件，程序保存地址由 bootloader 程序指定）  
    `fromelf --bin --output "$L@L.bin" "#L"`  
    ![](https://img-blog.csdnimg.cn/direct/fdaa8fcfa43c4e03a247225e90802b8b.png)
    
4.  app 程序
    

![](https://img-blog.csdnimg.cn/direct/d94fe0b667a4416ba59d7aff74394c24.png)

至此，App 程序就写完了，编译完成之后可以在工程的输出文件夹中看到编译生成的 bin 文件。  
使用同样的方法可以创建两个 App，分别为 app1、app2 两个的效果可以不一样。

这里设置 app1 输出：app_1 run，app2 输出：app_2 run。

#### 2.1.3 测试

测试步骤

1.  先把 Bootloader 程序烧录到芯片中
2.  复位运行，可以看到 bootloader 程序的提示信息
3.  使用串口助手传输 app 程序，传输完成之后按下按键，等待写入完成之后自动跳入 app 程序开始运行
4.  如果一直没有按键按下，5 秒之后直接跳入 app 开始运行

尝试更换两个 app，查看不同效果。  
![](https://img-blog.csdnimg.cn/direct/a028f66febf3469f8b0b1db94ef260f3.png)

**代码汇总地址：https://gitee.com/HzoZi/bootloader**

### 2.2 方式二：其他接口 / USB 拖拽等（未完成）