---
title: 从 STM32 到 Linux 服务器：基于 FreeRTOS 与 ESP-01S 的物联网采集终端
excerpt: 复盘一个集蓝牙配网、双通道采集、无线传输和服务端存储于一体的物联网终端。
date: 2025-03-25 11:09:21 +0800
categories:
  - 嵌入式
  - Linux
  - 物联网
tags:
  - STM32F103
  - FreeRTOS
  - ESP-01S
  - Socket
  - ADC
  - 蓝牙配网
---

## 项目背景

这个项目实现的是一套小型物联网数据采集系统：STM32 周期性读取两路模拟量，通过 ESP-01S 接入 Wi-Fi，将采集结果上传到部署在 Linux 服务器上的程序；设备第一次使用时，可以通过蓝牙接收 Wi-Fi 名称和密码，成功配网后再把凭据保存到片内 Flash，避免每次重启都重新配置。

它不只是“读取 ADC 再发串口”这么简单。真正需要解决的问题包括：多个串口如何并行接收数据、AT 指令失败后如何恢复、网络配置如何持久化、设备运行状态如何反馈，以及上传流程卡死后怎样让系统自动回到可用状态。

最终系统由两部分组成：

- STM32F103C8 终端：FreeRTOS、双通道 ADC、蓝牙串口、ESP-01S、片内 Flash、LED、蜂鸣器和独立看门狗。
- Linux 服务端：C 语言 Socket 服务，使用 `select` 同时管理监听套接字和客户端连接，将上报数据追加到文件，并提供读取和清空接口。

数据链路可以概括为：

```text
模拟传感器 -> STM32 ADC -> ESP-01S -> TCP/IP -> Linux 服务端 -> data.txt
                         ^
                    蓝牙首次配网
```

## 为什么选择 FreeRTOS

设备需要同时处理数据采集与上传、串口接收、状态灯闪烁和异常告警。如果全部写进一个裸机大循环，等待 AT 指令返回时，灯光提示等逻辑也会被阻塞，后续扩展会越来越困难。

因此项目使用 FreeRTOS 将核心工作拆成两个任务：

- `mainTask` 负责检查联网状态、读取或写入 Wi-Fi 配置、采集 ADC 数据并上传。
- `tipsTask` 负责根据队列中的状态值控制绿灯、黄灯、红灯和蜂鸣器。

主函数只负责完成硬件初始化、创建任务并启动调度器：

```c
int main(void) {
    Init();

    xTaskCreate(mainTask, "mainTask",
                configMINIMAL_STACK_SIZE * 8, NULL, 3, NULL);
    xTaskCreate(tipsTask, "tipsTask",
                configMINIMAL_STACK_SIZE, NULL, 3, NULL);
    vTaskStartScheduler();

    while (1);
}
```

这里的关键不是任务数量，而是职责边界。网络模块只需要向 `tipsQueue` 发送 `1`、`0` 或 `-1`，提示任务便能分别表现“执行中”“执行成功”和“执行失败”。这样 AT 指令处理不需要直接管理提示灯的闪烁节奏。

## 串口中断与队列：把接收和解析分开

ESP-01S 通过 USART2 工作在 115200 波特率，蓝牙模块通过 USART3 工作在 9600 波特率。两个串口都采用中断接收，ISR 只读取一个字节并写入 FreeRTOS 队列，完整响应的拼接和判断留在任务上下文完成。

以 Wi-Fi 模块为例：

```c
void USART2_IRQHandler(void) {
    if (USART_GetITStatus(USART2, USART_IT_RXNE) == SET) {
        USART_ClearITPendingBit(USART2, USART_IT_RXNE);
        char data = USART_ReceiveData(USART2);

        BaseType_t stat = pdTRUE;
        xQueueSendFromISR(queueWIFI, &data, &stat);
    }
}
```

任务侧的 `command()` 在发送 AT 指令前清空旧消息，然后持续从 `queueWIFI` 取数据，根据响应末尾的 `OK`、`FAIL`、`ERROR` 或 `busy p...` 判断结果，同时用系统节拍实现超时退出。

这种设计比在中断里等待完整字符串更稳妥：中断处理保持短小，复杂字符串解析不会长时间阻塞其他中断。不过当前实现还可以进一步补上缓冲区上限检查，并按 FreeRTOS 推荐方式处理 `xHigherPriorityTaskWoken` 和必要的任务切换。

## 首次蓝牙配网与 Flash 持久化

设备启动后先查询 ESP-01S 当前是否已连接接入点，再检查片内 Flash 是否保存过配网信息，随后分三种情况处理：

1. 已联网：直接进入采集上传循环。
2. 未联网但 Flash 有配置：读取配置并自动重连。
3. 未联网且 Flash 无配置：等待手机通过蓝牙发送 Wi-Fi 名称和密码。

蓝牙消息使用 `!账号=密码!` 作为简单帧格式。程序找到两个 `!` 后结束接收，再由 `netWorkCommand()` 拆出账号和密码，拼成 ESP-01S 的 `AT+CWJAP_DEF` 指令。连接成功后，配置被写入 STM32 的 `0x0800F000` 地址。

```c
void setFlash(char *buffer) {
    flashClean();

    changeFlashFor32(FLASH_ADDRESS, 0);
    for (int i = 0; i < FLASH_STRING_BUF_LEN; i += 4) {
        changeFlashFor32(FLASH_ADDRESS + 4 + i,
                         *(int *)(buffer + i));
    }
}
```

Flash 页开头的状态字用于判断配置是否存在，后续区域保存固定 40 字节的字符串。复位按键短按会直接重启，长按则先擦除 Flash 再重启，从而恢复到首次配网状态。

这个方案实现简单，适合资源受限的单片机，但工程化版本还应增加长度校验、格式校验和校验和，避免异常掉电或无效蓝牙数据留下错误配置。Wi-Fi 密码也不应长期以明文形式保存。

## 双通道 ADC 与数据封装

采集端使用 ADC1 的通道 1 和通道 4，对应 PA1 和 PA4。每次读取前动态配置规则通道，软件触发转换并等待 EOC 标志：

```c
uint16_t AD_GetValue(uint8_t ADC_Channel) {
    ADC_RegularChannelConfig(ADC1, ADC_Channel, 1,
                             ADC_SampleTime_55Cycles5);
    ADC_SoftwareStartConvCmd(ADC1, ENABLE);
    while (ADC_GetFlagStatus(ADC1, ADC_FLAG_EOC) == RESET);
    return ADC_GetConversionValue(ADC1);
}
```

两路原始值会被整理为 `parameter1:数值,parameter2:数值`，再放入 `stm32_msg` 字段。随后 ESP-01S 依次建立 TCP 连接、进入透传模式、发送数据、退出透传并关闭连接。

项目当前每采集一次就建立一次 TCP 连接，逻辑直观，也便于服务端按短连接处理；代价是 AT 指令交互较多、上传延迟和功耗较高。如果采样频率提高，更合适的方案是维持长连接，并增加心跳、断线检测和指数退避重连。

## Linux 服务端如何接收数据

服务端使用原生 Socket API 监听 `0.0.0.0:8000`，并通过 `select` 处理多个连接。它约定了三个路径：

- `/0`：接收 STM32 上报，从请求中提取 `stm32_msg` 并追加到 `data.txt`。
- `/1`：读取历史记录，拼成 JSON 数组返回给客户端。
- `/2`：清空数据文件并返回空数组。

相较于“一连接一线程”，`select` 在这个规模下不需要额外线程同步，代码也更容易理解。服务端将数据落到普通文本文件，适合演示和快速验证完整链路；当数据量增大时，可以替换为 SQLite 或时序数据库，并保留相同的设备上报接口。

现有 Keil 构建日志显示固件使用 ARMCC 5.06 编译，最终代码区约 20 KB，并达到 `0 Error(s), 0 Warning(s)`。这说明当前工程能够完成编译链接，但编译通过并不代表通信边界已经足够健壮。

## 可靠性设计：重试、提示和看门狗

无线通信最常见的问题不是永久失败，而是暂时无响应、模块忙或网络断开。项目中的每条关键 AT 指令都设置了超时，并在失败后等待一段时间重试。与此同时，`tipsTask` 会给出可观察的状态：

- 绿灯表示正常工作。
- 黄灯闪烁表示正在执行指令。
- 红灯和间歇蜂鸣表示指令失败并等待重试。

进入正式采集循环后，独立看门狗开始工作。程序只在一轮采集上传开始前喂狗；如果网络流程长时间卡住，系统将自动复位。这个策略把“网络上传链路是否能持续完成”直接作为设备健康条件，比在多个函数里无条件喂狗更有意义。

## 复盘：当前实现还需要修正什么

这套代码已经串起完整链路，但从演示原型走向稳定运行，还需要处理几个明确的问题。

### 1. 蓝牙回包使用了错误的串口

`BlueTooth.c` 中蓝牙接收使用 USART3，但 `blueToothSendByte()` 实际调用的是 USART2。这样配网成功后的回包会被发给 ESP-01S，而不是蓝牙模块。应把发送寄存器和状态检查统一改为 USART3。

### 2. 服务端解析缺少边界检查

`dealRequset()` 直接对 `strstr()` 的结果再次搜索。如果请求缺少 `stm32_msg` 或结束引号，空指针会导致进程崩溃。读取 `data.txt` 时也没有判断 `fopen()` 是否成功。更稳妥的实现应先验证请求行和正文长度，再用明确的数据格式解析，并对所有文件操作检查返回值。

### 3. `select` 的 `nfds` 参数被写死

当前代码调用 `select(100, ...)`。Linux 要求第一个参数是“当前最大文件描述符加一”，它与最多保存 100 个连接不是同一个概念。当系统分配的 fd 大于等于 100 时，该连接不会被检查。服务端应该维护 `max_fd`，并限制连接数组边界。

### 4. 中断服务函数中不应长时间延时

复位按键的 EXTI0 中断通过循环和 `Delay_ms(20)` 判断长按，这会让处理器长时间停留在中断上下文，影响串口接收和 RTOS 调度。更好的办法是中断中只记录按下事件，再由任务或软件定时器完成消抖和长按计时。

### 5. 传输协议需要明确化

终端发送的是带正文的 GET 请求，但服务端返回内容只有状态行和正文，没有 `Content-Type`、`Content-Length` 等标准头部，也没有对 `/0` 明确回包。演示环境可以工作，长期维护时应选择一种清晰协议：要么实现规范 HTTP，请求正文使用 JSON；要么直接定义带长度字段的 TCP 帧，避免依赖字符串搜索判断消息边界。

## 总结

这个项目把嵌入式开发中几个常见但容易割裂的知识点放进了同一条真实链路：ADC 数据采集、FreeRTOS 任务和队列、串口中断、蓝牙配网、片内 Flash、ESP-01S AT 指令、TCP 通信、Linux I/O 多路复用以及看门狗恢复。

对我来说，最大的收获不是“让两路数据出现在服务器文件里”，而是认识到一个联网设备必须同时考虑正常流程和失败流程。能上传只是第一步；知道何时超时、如何反馈、怎样恢复，以及如何保证协议和缓冲区边界，才是从功能样机走向可靠系统的关键。
