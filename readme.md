
## 概述

### 软件环境

 - Toolchain/IDE : MDK-ARM V5
 - package version: STM32Cube FW_F4 V1.16.0
 - FreeRTOS version: 9.0.0
 - CMSIS-RTOS version: 1.02

### 编程规范

- 变量和函数命名方式遵循 Unix/Linux 风格
- 在 Application/AppCtrl 中都为强实时的控制任务，禁止使用任何阻塞操作
- 在其他任务中使用的阻塞操作要设置合理的超时时间，优先级、频率不同的任务谨慎使用互斥锁等阻塞操作，防止优先级翻转，如频率较低的 detect_task 和频率较高的 info_get_task

### 注意事项

- 为了防止出现中文字符乱码，所有新建文件请使用 UTF-8 格式


- sys_config.h 文件包含了整个步兵车系统的配置参数，可以按照需求更改相应参数


### 模块离线说明

当车辆的某个模块离线时，可以根据开发板指示灯和蜂鸣器的不同状态判断哪个模块出现了问题，并进行错误定位

目前按照离线模块的优先级进行错误指示，例如云台电机优先级高于拨弹电机，如果同时发生离线，先指示当前离线设备是云台电机

模块离线对应的状态如下，数字对应红灯每次连续闪的次数，按照优先级排序：

1. 云台 pitch 轴电机离线
2. 云台 yaw 轴电机离线
3. 底盘单轴陀螺仪传感器离线
4. 拨弹电机离线
5. 底盘电机存在离线
6. 遥控器离线，裁判系统或者 PC 端串口没有连接，此时红灯常亮

### 文档

- 协议文档  [protocol](Doc/protocol/ch/readme.md)
- en doc [document](Doc/en/readme.md)

## 快速开始

### 硬件接口

主控板使用 RM 开发板 A 型，各个功能接口的位置如下：

![](Doc/ch/image/main_board_interface.PNG)

### 功能模块

#### 手动模式：

提供遥控器、鼠标键盘模式的基础控制。

注意：如果在 `sys_config.h` 文件中定义了 `AUTO_NAVIGATION` 宏，将开启 debug 和 全自动模式。

#### 全自动模式：

这种模式下底盘、云台、发射机构受到上层 PC 的完全控制，完全控制包含对这些执行机构以具体物理单位的控制，也包含对这些执行机构的模式切换。

#### 操作档位说明：

##### 手动档

遥控器控制：拨杆右上

- 开、关摩擦轮
- 单发、连发射击

鼠标键盘控制：（拨杆左中、右上）

- 开（Q）、关摩擦轮（Q + shift）
- 单发（单击左键）、连发射击（长按左键）
- 扭腰躲避（E）

##### Debug 档

机器人在调试击打装甲、自身躲避时使用（拨杆右中），此模式下摩擦轮继承手动时的开关状态

左拨杆位置对应功能：

- 上：底盘躲避，云台受遥控器手动控制
- 中：底盘躲避，云台跟随装甲板，此时云台和底盘都不受手动控制
- 下：底盘受遥控器手动控制，云台跟随装甲板

##### 自动档

正常比赛时使用（拨杆右下）

左拨杆位置对应功能：

- 上：摩擦轮打开，其他全接管
- 中：摩擦轮关闭，其他全接管
- 下：失能模式，底盘云台断电


## 工作模式说明

*备注：*

只有黑体部分为自动模式下能用到的各模块工作模式，其他工作模式不能在自动模式下切换。

### 云台

```c
typedef enum
{
  GIMBAL_RELAX         = 0,
  GIMBAL_INIT          = 1,
  GIMBAL_NO_ARTI_INPUT = 2,
  GIMBAL_FOLLOW_ZGYRO  = 3,
  GIMBAL_TRACK_ARMOR   = 4,
  GIMBAL_PATROL_MODE   = 5,
  GIMBAL_SHOOT_BUFF    = 6,
  GIMBAL_POSITION_MODE = 7,
  GIMBAL_RELATIVE_MODE = 8,
} gimbal_mode_e;
```

| 云台模式                 | 对应功能                        |
| ------------------------ | ------------------------------- |
| GIMBAL_RELAX             | 云台断电                        |
| GIMBAL_INIT              | 云台由断电状态回中              |
| GIMBAL_NO_ARTI_INPUT     | 无手动控制信息输入模式          |
| GIMBAL_FOLLOW_ZGYRO      | 云台正常的手动控制              |
| GIMBAL_TRACK_ARMOR       | 追踪装甲，保留                  |
| **GIMBAL_PATROL_MODE**   | 巡逻模式，云台 yaw 保持周期运动 |
| GIMBAL_SHOOT_BUFF        | 打大符模式，保留                |
| **GIMBAL_POSITION_MODE** | 相机在底盘上时使用              |
| **GIMBAL_RELATIVE_MODE** | 相机在云台  yaw 轴上时使用      |

### 底盘

```c
typedef enum
{
  CHASSIS_RELAX          = 0,
  CHASSIS_STOP           = 1,
  MANUAL_SEPARATE_GIMBAL = 2,
  MANUAL_FOLLOW_GIMBAL   = 3,
  DODGE_MODE             = 4,
  AUTO_SEPARATE_GIMBAL   = 5,
  AUTO_FOLLOW_GIMBAL     = 6,
} chassis_mode_e;
```

| 底盘模式                     | 对应功能                 |
| ------------------------ | -------------------- |
| CHASSIS_RELAX            | 底盘断电                 |
| CHASSIS_STOP             | 底盘停止/刹车              |
| MANUAL_SEPARATE_GIMBAL   | 手动模式底盘云台分离           |
| MANUAL_FOLLOW_GIMBAL     | 手动模式底盘跟随云台           |
| **DODGE_MODE**           | 底盘躲避模式，底盘固定旋转，平移不受控制 |
| **AUTO_SEPARATE_GIMBAL** | 底盘和云台分离模式，旋转、平移受上层控制 |
| **AUTO_FOLLOW_GIMBAL**   | 底盘跟随云台，平移受上层控制       |

### 发射机构

```c
typedef enum
{
  SHOT_DISABLE       = 0,
  REMOTE_CTRL_SHOT   = 1,
  KEYBOARD_CTRL_SHOT = 2,
  SEMIAUTO_CTRL_SHOT = 3,
  AUTO_CTRL_SHOT     = 4,
} shoot_mode_e;
```

| 发射机构模式             | 对应功能                 |
| ------------------ | -------------------- |
| SHOT_DISABLE       | 发射机构断电               |
| REMOTE_CTRL_SHOT   | 遥控器控制发射机构            |
| KEYBOARD_CTRL_SHOT | 键盘控制发射机构             |
| SEMIAUTO_CTRL_SHOT | 单发、连发上层控制            |
| **AUTO_CTRL_SHOT** | 摩擦轮开关、速度、单发、连发全部上层控制 |

为了击打装甲时响应更及时，在这个版本的程序中，自动模式时的发弹还是由底层触发。

## 程序说明

### 程序体系结构

#### 体系框架

1. 使用免费及开源的 freertos 操作系统，兼容其他开源协议 license；
2. 使用标准 CMSIS-RTOS 接口，方便不同操作系统或平台间程序移植；

#### 内部结构

1. 多任务环境，相比传统步兵控制框架，可以直接实现多线程逻辑，以及阻塞任务；
2. 较为完备的底层上层通信协议，实现上层对步兵各个机构模块的反馈信息获取和控制；
3. 内部程序模式切换、数据、控制各个任务隔离处理，方便添加和裁剪功能；
4. 底盘、云台、发射等机构，内部的控制模式去耦合，便于不同需求控制时的模式切换；

#### 程序框架

1. 基于 HAL 库的 BSP 层，主要提供can、uart、spi、flash、io等板级的配置和通信接口；
2. 数据交互层，这里是程序中会调用 BSP 层函数唯一的地方，主要为应用程序和硬件设备的数据交互；
3. 通信层，负责数据和 uart 硬件间发送和接收，以及这些数据的打包、解析，包含协议部分；
4. 信息获取，交互层可以直接使用的数据、通信层解析好的数据，在这个任务中获取后转化为反馈和控制信息；
5. 模式切换，在不修改控制架构的情况下，实现现有机构功能模式的自定义组合唯一需要修改的地方；
6. 控制任务，云台、底盘、射击这三个机构的控制，包含这些机构的示例模式。

### 软件体系

#### 程序启动时序

各个任务的启动时序图

![](Doc/ch/image/startup_sequence.PNG)

### 硬件体系

1. 主控 MCU：STM32F427IIHx，配置运行频率180MHz
2. 模块通信方式：CAN；CAN设备：电机电调、陀螺仪模块
3. 上下层通信方式：uart
4. 麦轮安装方式，X型

#### 硬件连接框图

![](Doc/ch/image/hardware_structure.PNG)

### 协议数据

#### 数据分类

协议数据按照通信方向可以分为两大类：

底层发送给上层的数据：

1. 反馈信息：包含各个机构传感器反馈信息、底层计算出来的一些反馈信息；
2. 底层状态信息：包含底层设备运行状态、底层对上层数据的一些响应等；
3. 转发数据：包含裁判系统的全部信息、服务器端的自定义信息；

底层接收的上层数据：

1. 控制信息：上层对底层 3 个执行机构的控制信息；
2. 配置信息：上层对机器人如轮距、轴距、云台位置等结构配置信息，以及上层程序运行状态，底层响应级别等；
3. 转发数据：需要底层转发给裁判系统，并最终在客户端上显示出来的用户自定义信息；

#### 数据流图

![](Doc/ch/image/data_flow.PNG)

