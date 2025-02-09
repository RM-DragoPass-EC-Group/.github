# 开发板C
步兵一个，哨兵底盘、云台各一个。
## CAN
### 介绍
连接电机电调的总线，有CAN1和CAN2两条线。在开发板上CAN1有两个接口，2pin，接口定义为（RX, TX）。 CAN2有两个接口，4pin，接口定义为（5V, GND, RX, TX）。详见[用户手册](https://github.com/RM-DragoPass-EC-Group/.github/blob/main/Docs/%E5%BC%80%E5%8F%91%E6%9D%BF/RoboMaster%20%20%E5%BC%80%E5%8F%91%E6%9D%BF%20C%20%E5%9E%8B%E7%94%A8%E6%88%B7%E6%89%8B%E5%86%8C.pdf)。**注意一个设备的TX连接对方设备的RX.** 总线id范围为1-8。CAN1的2pin接口非常脆弱，请勿用力插拔。

### 接线
每个总线最多挂载8个电调设备。为保证信号完整性，尽量少挂载电机（将电机均匀分配在CAN1和CAN2上）。目前为方便走线，步兵云台上的所有电机（发射机构电机+pitch轴电机）连接CAN2总线，底盘所有电机（四个底盘电机+yaw轴电机）连接CAN1总线。哨兵底盘每侧的电机在一个CAN总线上，云台发射机构为CAN1，云台电机为CAN2.

### 开发
EventOS有内置的电机初始化抽象，可以将电机定义在任意总线的任意id上。另外哨兵需要通过CAN总线进行云台、底盘开发板版间通信，待开发。

## Micro-USB
USB形态的UART串口，用来连接上位机NUC. 收发包已经成熟，所有交互在Robo_usb.c中完成，数据结构在Robo_usb.h中定义。这个接口非常脆弱，请勿用力插拔。

## UART
串口，3pin，接口定义为（RX, TX, GND），用于连接裁判系统。数据传输已经成熟，可以在Robo_get_messege()全局变量中获取裁判系统结构体实时数据。

## DBUS
连接遥控器接收机，3pin. 由于接收机功率较大，**必须连接24V电池使用**。这个接口比较脆弱，在大功率下接触不良容易打火，注意固定。

## SWD
连接烧录器/debugger，4pin，接口定义为（5V, GND, CLK, DATA），目前使用ST-Link有线连接，未来可能换用无线烧录器。

## XT30
供电接口，共四个。一个为输入母口，三个为输出公口。24V电压。

# 中心板
中心板是CAN总线&供电二合一分线板。步兵三个（底盘、云台、发射机构），哨兵五个（底盘两个、云台、发射机构、pc+雷达）。

## 供电
一路XT60输入，引出六路XT30输出。请勿将XT30用作输入。

## 信号
**注意注意注意：每个中心板只能有一路CAN总线输入，否则一定会出现信号问题。**
在顶部使用2pin或4pin的CAN总线输入，引出6路2pin的CAN信号。注意从开发板输出之后，2pin/4pin与CAN1和CAN2无关，因为CAN2 4pin接入中心板后变成2pin. 建议不要将侧面CAN接口用作输入，可能出现信号问题。

## 接线
### 步兵
#### 底盘中心板
供电：电源管理模块底盘供电(IN), 底盘3508*4(OUT)

信号：开发板CAN1(IN), 底盘3508*4 + yaw6020(OUT)

#### 云台中心板
供电：电源管理模块云台供电(IN), yaw6020 + pitch6020(OUT)

信号：开发板CAN2(IN), pitch6020 + 摩擦轮3508*2 + 拨弹轮2006(OUT)

#### 发射机构中心板
供电：电源管理模块发射机构供电(IN), 摩擦轮3508*2 + 拨弹轮2006(OUT)

信号：无

### 哨兵
#### 底盘中心板1（电源管理模块同侧）
供电：电源管理模块底盘供电(IN), 舵机6020\*2 + 轮组3508\*2 + 底盘中心板2(OUT)

信号：底盘开发板CAN2(IN), 舵机6020\*2 + 轮组3508\*2(OUT)

#### 底盘中心板2（电源管理模块异侧）
供电：底盘中心板1(IN), 舵机6020\*2 + 轮组3508\*2(OUT)

信号：底盘开发板CAN1(IN), 舵机6020\*2 + 轮组350\**2(OUT)

#### 云台中心板
供电：电源管理模块云台供电(IN), yaw6020 + pitch6020(OUT)

信号：云台开发板CAN2(IN), yaw6020 + pitch6020(OUT)

#### 发射机构中心板
供电：电源管理模块发射机构供电(IN), 摩擦轮3508*2 + 拨弹轮2006(OUT)

信号：云台开发板CAN1(IN), 摩擦轮3508*2 + 拨弹轮2006(OUT)

#### PC+雷达中心板
供电：电源管理模块PC供电(IN), PC变压器 + 激光雷达(OUT)

信号：无

# 电机
## 3508
高性能轮组电机，需要搭配C620电调使用。电机与电调之间通过7pin排线和3pin电源线连接。排线十分脆弱，请注意在接线时是否会被电机/轮组摩擦。另外摩擦轮使用的是拆除行星减速箱的3508电机。

电调有一个XT30电源线，一个CAN 2pin接口，一个PWM接口。目前所有电机均使用CAN信号控制电流，无需接PWM接口。电源最多输入16000mA，注意在PID控制器MAXOUT中设置16000.

电调正面有一个指示灯，行为详见[C620电调说明书](https://github.com/RM-DragoPass-EC-Group/.github/blob/main/Docs/%E7%94%B5%E8%B0%83%EF%BC%8DC620%E8%AF%B4%E6%98%8E%E4%B9%A6.pdf)

### 更改CAN ID
使用尖镊子、SIM卡针等尖锐物品捅进C620电调侧面的孔洞。由于电调内部的按键面积小于孔洞，且不一定在孔洞正中间，可能第一时间戳不到，可以尝试各种角度。如果戳成功，会有明显的按键触感反馈。电调在通电状态下戳一次进入设置ID模式，之后连续戳N次将ID设置为N.

## 6020
空心关节电机，内置电调。有一个XT30电源线，一个CAN 2pin接口，一个PWM接口。电源最多输入30000mA，注意在PID控制器MAXOUT中设置30000.

电机侧面有一个指示灯，行为详见[6020电机说明书](https://github.com/RM-DragoPass-EC-Group/.github/blob/main/Docs/%E7%94%B5%E6%9C%BA-GM6020%E8%AF%B4%E6%98%8E%E4%B9%A6.pdf)

### 更改CAN ID
在断电时使用尖锐物品拨动电机底面的CAN ID开关。共有四个开关，前三个代表二进制CAN ID的1，2，3位，靠近电机中心为0，远离点击中心为1，ID=0时无效，因此6020电机ID范围为1-7. 最后一个开关时CAN电阻，无需调整。

## 2006
拨弹轮电机，需要搭配C610电调使用。电机与电调之间通过7pin排线和3pin电源线连接。**排线十分脆弱，请注意在接线时是否会被电机/轮组摩擦。**

电调有一个XT30电源线，一个CAN 2pin接口，一个PWM接口。

电调正面有一个指示灯，行为详见[C610电调说明书](https://github.com/RM-DragoPass-EC-Group/.github/blob/main/Docs/%E7%94%B5%E8%B0%83%EF%BC%8DC610%E8%AF%B4%E6%98%8E%E4%B9%A6.pdf)

### 更改CAN ID
C610电调上有CAN ID按键，通电后按下进入ID设置模式，按N次将ID设置为N.

# 裁判系统
电源管理模块是接线的中心模块，所有裁判系统接线从这里出发。

## 机器人电源
由于比赛规则要求底盘、云台、发射机构、minipc电源分控，这四路电源接在电源管理模块不同接口。均为XT30输出。

### 底盘电源
接入底盘中心板，为四个底盘电机（步兵）/四个轮组电机+四个舵机（哨兵）供电。在罚下时断电，会受到底盘功率限制。

### 云台电源
接入云台中心板，为云台pitch轴、yaw轴电机供电。在罚下时断电。

### 发射机构电源
接入发射机构中心板，为两个摩擦轮电机和拨弹盘电机供电。在死亡或无剩余发弹量时断电。

### PC电源
步兵直接接入PC变压器，哨兵接入PC+雷达中心板。不会断电。

### 发射机构电源
接入发射机构中心板，

## 主控模块接口
黑色航空线公口。由于主控模块也是公口，需要母对母的延长线连接。

## 装甲板接口
6pin，从电源管理模块出发，菊花链串联每块装甲板，最终回到电源管理模块，形成闭环。

## UART串口
3pin，连接开发板对应的UART串口。注意本设备的TX连接对方设备的RX.

## RFID接口
4pin，连接场地交互模块。

## 测速模块接口
航空线，连接测速模块。其中步兵有一个测速模块，哨兵有两个测速模块。

## 图传模块接口
航空线，连接图传模块。

## 超级电容接口
私有接口，连接超级电容模块，暂时未设计超级电容。

# 工业相机
一个Micro-USB 3.0接口，连接NUC.

# 激光雷达
一个XT30供电，连接PC+雷达中心板；一个RJ45信号线，连接NUC.

# 导电滑环
可以360°无限旋转的电线接口。

## 步兵接线
供电：云台中心板供电，发射机构供电，PC供电（底盘到云台），yaw轴电机供电（云台到底盘）

信号：CAN1 2pin（云台到底盘），测速模块航空线，图传模块航空线，裁判系统UART串口（底盘到云台）

## 哨兵接线
供电：云台中心板供电，发射机构供电，PC供电（底盘到云台），yaw轴电机供电（云台到底盘）

信号：板间通信CAN1 2pin（双向），yaw轴电机信号（云台到底盘），测速模块航空线（底盘到云台）