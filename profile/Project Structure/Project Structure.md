# 程序结构
在系统抽象的基础上，我们搭建了chassis、gimbal、shoot、control、USB和AA六个进程。其中chassis进程负责底盘控制，gimbal进程负责云台控制，shoot进程负责射击控制，control进程负责外设控制，USB进程负责与上位机通信，AA进程负责自瞄。

每个进程都有一个Init函数和一个Task函数。Init函数在进程启动时执行一次，Task函数在进程运行时循环执行。

## Chassis （Robo_Chassis.c/.h）
底盘进程负责从全局变量中获取预期的底盘移动速度，然后利用电机抽象使用电流控制电机。

底盘进程拥有一个句柄结构体"Task_Chassis_t chassis"，其中包含了底盘控制所需的所有参数。

### Init
流程如下：
1. 初始化底盘硬件参数（轮距、轮半径、电机减速比）
2. 初始化电机
3. 初始化PID（电机PID，底盘跟随PID，功率控制PID）
4. 初始化裁判系统
*底盘跟随是底盘朝向跟随云台朝向的运动模式*
*为了实现更好的机动，底盘的最大速度由功率限制和当前功率自动控制*
*由于步兵只有底盘的功率限制会利用裁判系统数据，所以步兵的所有裁判系统调用均在底盘进程中实现*
*裁判系统初始化会自动绑定中断并不断更新数据，不需要我们自己接收数据*

### Task
Task函数流程类似OOP的类方法，首先获取数据，然后处理数据，最后输出数据。

流程如下：
1. 从全局变量中获取预期底盘运动速度
2. 如果在底盘跟随模式，使用PID控制获得底盘额外旋转速度
3. 将底盘的x(前+后-), y(左+右-), w(逆时针+顺时针-)速度转换为左前、右前、左后、右后四个电机速度
4. 使用PID通过速度控制电机电流
5. 发送电流

## Gimbal （Robo_gimbal.c/.h）
云台进程负责从全局变量中获取预期的云台朝向，然后利用电机抽象使用电流控制电机。

云台进程拥有一个句柄结构体"Task_Gimbal_t gimbal"，其中包含了云台控制所需的所有参数。

### Init
流程如下：
1. 初始化电机
2. 初始化PID（yaw和pitch电机各有速度环、角度环双环PID）

### Task
Task函数流程类似OOP的类方法，首先获取数据，然后处理数据，最后输出数据。

流程如下：
1. 从全局变量中获取预期云台朝向角度
2. 使用角度环PID通过角度控制电机速度
3. 使用速度环PID通过速度控制电机电流
4. 发送电流

## Shoot （Robo_Shoot.c/.h）
射击进程负责从全局变量中获取预期的射击状态，然后利用电机抽象使用电流控制电机。

射击进程拥有一个句柄结构体"Task_Shoot_t shoot"，其中包含了射击控制所需的所有参数。

### Init
流程如下：
1. 初始化电机（两个摩擦轮3508电机，一个拨弹盘2006电机）
2. 初始化PID（摩擦轮电机PID，拨弹盘电机速度环PID，拨弹盘电机角度环PID）

### Task
Task函数流程类似OOP的类方法，首先获取数据，然后处理数据，最后输出数据。

Shoot代码中提供了打开摩擦轮电机、打开拨弹盘电机、关闭摩擦轮电机、关闭拨弹盘电机的方法，可以直接调用。

流程如下：
1. 从全局变量中获取预期射击状态
2. 如果射击状态为：关闭，关闭摩擦轮电机和拨弹盘电机；准备，打开摩擦轮电机；射击，打开摩擦轮电机和拨弹盘电机

## Control （Robo_Control.c/.h）
外设控制进程负责从中外设获取信息，然后在全局变量中发布底盘、云台预期状态。

### Init
所有全局变量tag置零。

### Task
使用遥控器两个开关的状态控制操作模式。一般左开关s1的下为机器人失能，中为遥控器控制，上为键鼠/自动控制。右开关s2模式可自定义，用于测试使用。

### Remote_Control
遥控器控制模式，使用遥控器控制底盘和云台。

底盘控制流程如下：
1. 获取遥控器左摇杆两个通道的值，计算底盘的前后和左右运动速度
2. 移动速度归一化（机器人向左前45°全速移动的速度是1，不是√2）
3. 在全局变量中发布底盘预期速度

云台控制流程如下：
1. 获取遥控器右摇杆两个通道的值，计算云台的yaw和pitch速度
2. 从全局变量中获取当前yaw和pitch角度，在此角度上增加offset
3. 依据机械结构进行俯仰限位
4. 在全局变量中发布云台预期角度

## USB （Robo_USB.c/.h）
USB进程负责与上位机通信，接收上位机发送的数据包，然后根据数据包类型执行相应的操作。

USB进程的发送数据包结构如下：
```c
typedef struct __attribute__((packed))
{
  uint8_t header;
  uint8_t detect_color : 1;  // 0-red 1-blue
  _Bool reset_tracker : 1;
  uint8_t reserved : 6;
  float roll;
  float pitch;
  float yaw;
  float aim_x;
  float aim_y;
  float aim_z;
  uint16_t checksum;
} ReceivePacket;	//相对于上位机
```
主要包含自瞄目标位置。

USB进程的接收数据包结构如下：
```c
typedef struct __attribute__((packed))
{
  uint8_t header;
  _Bool tracking : 1;
  uint8_t id : 3;          // 0-outpost 6-guard 7-base
  uint8_t armors_num : 3;  // 2-balance 3-outpost 4-normal
  uint8_t reserved : 1;
  float x;
  float y;
  float z;
  float yaw;
  float vx;
  float vy;
  float vz;
  float v_yaw;
  float r1;
  float r2;
  float dz;
  uint16_t checksum;
}SendPacket;
```
主要包含IMU数据。

### Init
1. 初始化USB
2. 初始化发送数据包

### Task
1. 接收
2. 发送

### receive
流程如下：
1. 设定单次传输数据包长度（只要大于数据包长度即可）
2. 调用USB_CDC抽象接收数据包
3. 检查数据包头
4. 检查CRC校验和

### send
流程如下：
1. 计算CRC校验和
2. 调用USB_CDC抽象发送数据包

## AA （Robo_AA.c/.h）
自瞄进程负责使用目标位置进行弹道解算，然后在全局变量中发布云台预期角度。

### Init
空

### Task
流程如下：
1. 将IMU和自瞄目标位置传入弹道解算函数
2. 在全局变量中发布云台预期角度