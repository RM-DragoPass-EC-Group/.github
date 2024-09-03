# Intro
目前我们所有的程序都基于仲恺的 EventOS 操作系统开发。这个操作系统提供了硬件、算法、数据类型、多进程和全局变量抽象，下面我将以步兵为例介绍这几种抽象的使用方法，并在此基础上讲解我们的程序结构。

# 抽象
## 一些常用的硬件抽象
### 电机
#### DJI_Motor_Init (Dji_Motor.c)
```c
/**
 * @brief 大疆电机初始化
 *
 * @param[in,out] motor 大疆电机句柄(不能为临时变量)
 * @param[in] motor_type 电机类型
 * @param[in] ID 电机设置的id
 * @param[in,out] can_handle can句柄
 * @return 电机句柄
 */
Motor_t *DJI_Motor_Init(Dji_Motor_t *motor, DJI_motor_type motor_type, uint16_t ID, Can_Handle_t *can_handle)
```
例如：
```c
motor = DJI_Motor_Init(dji_motor, DJI_Motor_C620, 1, &can1);
```
这行代码初始化了一个3508电机，在CAN1总线上的ID为1。

其中Dji_Motor_t是较为低级的电机句柄，在Dji_Motor.h中有如下定义：
```c
/**
 * @brief 大疆电机句柄
 */
typedef struct
{
	Motor_t motor;

	DJI_motor_type motor_type; /**< 电机类型 */
	DJI_Motor_tx_t tx_config;	 /**< 发送配置 */

	Soft_WitchDog_t witchdog; /**< 软件看门狗 */
} Dji_Motor_t;
```
Motor_t是较为高级的电机句柄，在Motor.h中有如下定义：
```c
/**
 * @brief 电机句柄结构体
 */
typedef struct
{
	float gear_ratio;
	float speed_gear;
	float position_gear;
	float position;									/**< 位置(rad) */
	float speed;								    /**< 速度(rad/s) */
	float torque;								    /**< 扭矩(N*m) */
	float inc_position;
	uint8_t temperature;						    /**< 温度 */
	void *config;								    /**< 电机配置 */
	Motor_Send_Data_t Send_Data;		            /**< 电机设置函数 */
	Motor_Send_Data_t Speed_Loop;		            /**< 速度环设置函数 */
	void *speed_loop_config;				        /**< 速度环设置句柄 */
	Motor_Send_Data_t Position_Loop;                /**< 位置环设置函数 */
	void *position_loop_config;			            /**< 位置环设置句柄 */
} Motor_t;
```
这个结构体中包含了绝大多数电机的状态信息，是硬件抽象的最高层。在使用电机是我们只需与这个结构体交互即可。

motor_type是电机型号。目前我们只使用到三种大疆电机：DJI_Motor_C620, DJI_Motor_C610和DJI_Motor_GM6020，分别对应3508, 2006和6020电机。

ID是电机在CAN总线上的ID.

can_handle是CAN总线句柄，只能是&can1或&can2.

#### Motor_Send_Data (Motor.c)
```c
/**
 * @brief 电机句柄发送函数
 * @param motor 电机句柄
 * @param data 发送数据
 */
void Motor_Send_Data(Motor_t *motor, float *data)
```
其中motor是电机句柄，data是发送的电流值，单位为毫安。例如：
```c
Motor_Send_Data(motor, 10000);
```
这行代码向motor发送了10000mA的电流。

#### 杂项 (Dji_Motor.h)
```c
/**
 * @brief 获取大疆电机位置(deg)
*/
#define Get_DJI_Motor_Position_Data(handle) Get_Motor_Position_Data(&(handle)->motor)
/**
 * @brief 获取大疆电机速度(deg/s)
*/
#define Get_DJI_Motor_Speed_Data(handle) Get_Motor_Speed_Data(&(handle)->motor)
/**
 * @brief 获取大疆电机转矩
*/
#define Get_DJI_Motor_Torque_Data(handle) Get_Motor_Torque_Data(&(handle)->motor)
/**
 * @brief 获取大疆电机温度
*/
#define Get_DJI_Motor_Temperature_Data(handle) Get_Motor_Temperature_Data(&(handle)->motor)
/**
 * @brief 获取电机控制数据最大值
 */
#define Get_DJI_Motor_Control_Max(handle) (dji_motor_control_max[(handle)->motor_type])
```

#### 杂项 (Motor.h)
```c
/**
 * @brief 获取电机位置(rad)
 */
#define Get_Motor_Position_Data(motor) ((motor)->position)
/**
 * @brief 获取电机增量位置(rad)
 */
#define Get_Motor_Inc_Position_Data(motor) ((motor)->inc_position)
/**
 * @brief 获取电机速度(rad/s)
 */
#define Get_Motor_Speed_Data(motor) ((motor)->speed)
/**
 * @brief 获取电机转矩
 */
#define Get_Motor_Torque_Data(motor) ((motor)->torque)
/**
 * @brief 获取电机温度
 */
#define Get_Motor_Temperature_Data(motor) ((motor)->temperature)
```

### IMU
从C板的IMU获取的所有数据均存储在AHRS结构体中，include "AHRS.h"即可使用。
例如：
```c
current_yaw = AHRS.euler_angle.yaw;
```
其中AHRS_t结构体在AHRS.h中定义如下：
```c
/**
 * @brief AHRS句柄
 */
typedef struct
{
	Euler_Data_t euler_angle;							/**< 欧拉角 */
	Accel_Data_t accel;									/**< 加速度 */
	Accel_Data_t real_accel;							/**< 惯性坐标系下的加速度 */
	Gyro_Data_t gyro;									/**< 角速度 */
	Mag_Data_t mag;										/**< 磁场 */
	float quat[4];										/**< 姿态四元数 */
	float g_norm;										/**< 标定时的重力加速度 */
	float sample_time;									/**< 采样时间 */
	Gyro_Data_t gyro_offset;							/**< 角速度零偏校正值 */
	AHRS_Calculate_fun_t AHRS_Calculate_fun; 			/**< AHRS解算算法 */
	void *IMU_handle;									/**< IMU句柄*/
	Get_IMU_fun_t Get_IMU_fun;							/**< 获取IMU数据函数 */
	void *mag_handle;									/**< 磁力计句柄*/
	Get_Mag_fun_t Get_Mag_fun;							/**< 获取磁力计数据函数 */
	uint8_t count;
} AHRS_t;
```

### 裁判系统
裁判系统的数据存储在Task_Chassis_t下的Reference_t结构体中，在Chassis_Task中可以直接使用。
例如：
```c
power_limit = chassis.referee.game_robot_status.chassis_power_limit;
```
Reference_t结构体的定义在reference.h中。由于内容很多，这里不引用，请各位需要时查看。

### 键盘
键盘句柄存储在RC_Ctrl_Key.c中的Keyboard_t结构体中，并且可以通过Button_Register方法绑定中断，触发事件。
其中Keyboard_t结构体定义如下：
```c
typedef struct 
{
	Button_t  key_W ;
	Button_t  key_S ;
	Button_t  key_A ;
	Button_t  key_D ;
	Button_t  key_Shift ;
	Button_t  key_Ctrl ;
	Button_t  key_Q ;
	Button_t  key_E ;
	Button_t  key_R ;
	Button_t  key_F ;
	Button_t  key_G ;
	Button_t  key_Z ;
	Button_t  key_X ;
	Button_t  key_C ;
    Button_t  key_V ;
	Button_t  key_B ;
	Button_t  mouse_press_left;
	Button_t  mouse_press_right;
	Key_Mode key;
} Keyboard_t;
```
大疆遥控器只能够透传以上按键状态，除此以外的按键不能使用。

Button_t结构体定义在Button.h中，如下：
```c
typedef struct
{
	uint8_t event : 4;									/**< 按键事件 */
	uint8_t status : 3;									/**< 按键状态 */
	uint8_t long_press_flag : 1;						/**< 是否为长按 */
	uint8_t active_level : 1;							/**< 按键真电平 */
	uint8_t clicks_count : 7;							/**< 点击次数 */
	uint8_t press_count;								/**< 按键滤波计数 */
	uint32_t press_down_time;							/**< 按下时间 */
	uint32_t press_up_time;								/**< 松开时间 */
	uint8_t (*get_level_fun)(void *);					/**< 获取按键电平函数 */
	void *get_level_conf;								/**< 获取按键电平配置 */
	void (*callback_fun[Button_Event_Num])(void *); 	/**< 按键回调函数 */
	void *next;											/**< 链表指针 */
	//SEML_Object
} Button_t;
```
Button_t功能十分强大，可以分辨长按时间、点击次数等高级事件。

#### Button_Init (Button.c)
```c
/**
 * @brief 按键初始化
 * @param button 按键句柄
 * @param get_level_fun 获取按键函数
 * @param get_level_config 获取按键配置
 * @param active_level 按键按下电平
 */
void Button_Init(Button_t *button, FunctionalState active_level, uint8_t (*get_level_fun)(void *), void *get_level_config);
```
例如：
```c
Button_Init(&keyboard->key_W,1,Get_Key_Status,(void*)RC_Key_W);
```
这行代码使用Get_Key_Status方法初始化了键盘的W键。

#### Button_Register (Button.c)
```c
/**
 * @brief 按键事件回调注册
 * @param button 按键句柄
 * @param event 按键事件
 * @param callback_fun 事件回调函数
 */
void Button_Register(Button_t *button, Button_Event_t event, Button_Callback_t callback_fun);
```
其中button是按键句柄，event是事件类型，callback_fun是绑定的回调函数。
例如：
```c
Button_Register(&keyboard->key_W,Press_Hold,Front_Press_Hold);
```
这行代码将Front_Press_Hold函数绑定为在键盘的W键长按时触发，接下来我们可以在Front_Press_Hold函数中编写我们的逻辑。

事件类型有以下几种：
```c
typedef enum
{
	Press_None = 0,		 	/**< 无按键事件(松开时候一直触发) */
	Press_Hold,				/**< 按键按下(按下时候一直触发) */
	Press_Up,				/**< 按键松开(每松开一次触发) */
	Press_Down,				/**< 按键按下(每按下一次触发) */
	Single_Clink,			/**< 单击按键 */
	Double_Clink,			/**< 双击按键 */
	Multiple_clicks,	 	/**< 多次点击(>2次) */
	Long_Press_Start,		/**< 按键长按开始 */
	Long_Press_Hold,	 	/**< 按键长按 */
	Long_press_Release 		/**< 按键长按释放 */
} Button_Event_t;
```

### 遥控器
从遥控器获取的所有数据均存储在RC_Ctrl结构体中，include "DR16_Remote.h"即可使用。结构体定义在DR16_Remote.h中。由于内容很多，这里不引用，请各位需要时查看。

例如：
```c
controller_ch1 = RC_Ctrl.rc.ch1;
```

### USB串口
所有USB行为均在Robo_USB.c中定义。其中USB_Task已经绑定为一个进程，随着程序运行会循环执行。该进程中有两个主要函数：receive和send. 其中send定义如下：
```c
static void send(ReceivePacket *packet)
{
	memcpy(usb_send, packet, sizeof(ReceivePacket)-2);
  	packet->checksum = get_CRC16_check_sum(usb_send, sizeof(ReceivePacket)-2, 0xffff);
	memcpy(usb_send, packet, sizeof(ReceivePacket));
	CDC_Transmit_FS(usb_send, sizeof(ReceivePacket));
}
```
其中ReceivePacket是相对于上位机的数据包结构体，由开发板发往上位机

receive定义如下：
```c
static void receive(void)
{
	uint32_t len = 1024;
	CDC_Receive_FS(usb_buf, &len); // Read data into the buffer
	if (usb_buf[0] == AUTO_AIMING_HEADER)
	{
		memcpy(&received_packet, usb_buf, sizeof(SendPacket));
	}

	else if (usb_buf[0] == NAVIGATION_HEADER)
	{
		memcpy(&received_navi, usb_buf, sizeof(SendNavi));
		navigation_handler();
	}
	else
	{
		navigation.vx = 0.0f;
		navigation.vy = 0.0f;
		navigation.vw = 0.0f;
	}
}
```
len需要大于接收数据包的长度，usb_buf是接收数据的缓冲区。我们将接收数据的第一位作为header来判断数据包类型，然后根据不同的header调用不同的处理函数。

## 一些常用的算法抽象
### 常用计算
常用计算函数在math_common.c中定义，如指数、对数、三角函数、阶乘等。需要时可以查看。

### PID
#### PID_Init (PID.c)
```c
/**
 * @brief PID配置初始化
 *     - 只配置最基础的pid参数
 *     - 抗积分饱和方式：动态钳位
 *     - 微分低通滤波:不滤波
 * @param[out] PID_Config PID配置结构体指针
 * @param[in] Kp 比例因子
 * @param[in] Ki 积分因子
 * @param[in] Kd 微分因子
 * @param[in] PID_Max 输出最大限幅值
 * @param[in] PID_Min 输出最小限幅值
 * @param[in] sample_time 采样时间(秒)
 */
void PID_Init(PIDConfig_t *PID_Config, PIDElem_t Kp, PIDElem_t Ki, PIDElem_t Kd, PIDElem_t PID_Max, PIDElem_t PID_Min, float sample_time)
```
例如：
```c
PID_Init(pid_config, 15, 0.5f, 0, 16000, -16000, 0.001f);
```
这行代码初始化了一个PID控制器，kp为15，ki为0.5，kd为0，输出最大值为16000，输出最小值为-16000，采样时间为1ms。

其中PIDElem_t是PID参数的数据类型，定义为float.

PIDConfig_t结构体在PID.h中定义如下：
```c
typedef struct
{
	PID_Param_t Param; 					/**< PID参数 */

	float SampTime; 					/**< 采样时间 */

	PIDElem_t PIDMax;					/**< PID最大值 */
	PIDElem_t PIDMin;					/**< PID最大值 */
	PIDElem_t ErrValue;					/**< 误差 */
	PIDElem_t lest_ErrValue;			/**< 上一次误差 */
	PIDElem_t ITerm;					/**< 积分累加值 */
	PIDElem_t DiffValue;				/**< 误差的导数 */
	PID_anti_windup_t anti_windup;		/**< 积分抗饱和方式 */
	PIDElem_t Ki_Clamp;					/**< 反馈抑制抗饱和系数 */
	PIDElem_t Kd_lowpass;				/**< 微分低通滤波器系数 */
	PID_Callback_Fun_t Callback_Fun;	/**< PID回调函数 */

	PIDElem_t PIDout; 					/**< PID输出值 */
} PIDConfig_t;
```
其中PID_Param_t在PID.h中定义如下：
```c
typedef struct
{
	PIDElem_t Kp; /**< 比例因子 */
	PIDElem_t Ki; /**< 积分因子 */
	PIDElem_t Kd; /**< 微分因子 */
} PID_Param_t;
```

#### Basic_PID_Controller (PID.c)
```c
/**
 * @brief 基础位置式PID控制器
 * @param[in,out] PIDConfig PID配置
 * @param[in] E_value 期望值
 * @param[in] C_value 实际值
 * @return 当前PID输出
 */
PIDElem_t Basic_PID_Controller(PIDConfig_t *PID_Config, const PIDElem_t E_value, const PIDElem_t C_value)
```
其中E_value是期望值，C_value是实际值。例如：
```c
PID_out = Basic_PID_Controller(pid_config, target_speed, current_speed);
```
这行代码计算了一个PID输出，期望值为target_speed，实际值为current_speed。

### CRC校验
get_CRC16_check_sum方法定义在CRC8_CRC16.c中，用于计算CRC16校验和。
```c
/**
  * @brief          计算CRC16
  * @param[in]      pch_message: 数据
  * @param[in]      dw_length: 数据和校验的长度
  * @param[in]      wCRC:初始CRC16
  * @retval         计算完的CRC16
  */
uint16_t get_CRC16_check_sum(uint8_t *pch_message,uint32_t dw_length,uint16_t wCRC)
```
其中pch_message是数据，dw_length是数据长度，wCRC是初始CRC16值。例如：
```c
checksum = get_CRC16_check_sum(msg, sizeof(msg), 0xffff);
```
这行代码计算了msg的CRC16校验和。

### 滤波器
#### RMS_filter_Init (math_filter.c)
```c
/**
 * @brief 均方根滤波器初始化
 * @param[out] RMS_filter  均方根配置结构体指针
 * @param[in] window_width 滑动窗口大小
 * @param[in] init_value 初始值
 * @param[in,out] buffer 队列缓存数组
*/
void RMS_filter_Init(RMS_filter_t *RMS_filter,const uint16_t window_width,const float init_value,float *buffer);
```
例如：
```c
RMS_filter_Init(&rms_filter, 10, 0, rms_buffer);
```
这行代码初始化了一个滑动窗口大小为10的均方根滤波器，初始值为0。

其中RMS_filter_t结构体在math_filter.h中定义如下：
```c
typedef struct 
{
    uint16_t window_width;  /**< 窗口大小 */
    float gain;
    uint16_t count;         /**< 计数值 */
    float RMS_value;        /**< rms值 */
    float square_sum;       /**< 平方和 */
    s_queue data_queue;     /**< 数据队列*/
} RMS_filter_t;
```

#### RMS_filter (math_filter.c)
```c
/**
 * @brief 均方根滤波器
 * @param[in,out] RMS_filter 滤波器指针
 * @param[in] data 输入数据
 * @return 滤波后结果
*/
_FAST float RMS_filter(RMS_filter_t *RMS_filter,const float data);
```
例如：
```c
filtered_data = RMS_filter(&rms_filter, raw_data);
```
这行代码对raw_data进行均方根滤波。

#### 其他滤波器
math_filter.c中还有滑动均值滤波器，使用方法类似。卡尔曼滤波器算法十分简单，但暂未封装。

### 其他算法
math目录下还有一些其他算法，如矩阵运算、向量运算等。需要时可以查看。

## 一些常用的数据类型抽象
### Queue
s_queue结构体定义在queue.h中，如下：
```c
typedef struct
{
	void *address;							/**< 队列初始地址*/
	uint16_t front;							/**< 队列头指针*/
	uint16_t rear;							/**< 队列尾指针*/
	uint16_t size;							/**< 队列大小*/
	uint16_t elem_size;						/**< 队列元素大小*/
	queue_full_hander_t full_hander : 2; 	/**< 满队处理方式*/
	uint8_t error_code : 4;					/**< 队列错误码*/
	uint8_t use_extern_buffer : 1;			/**< 使用外部缓存数组*/
	SEML_LockType_t Lock : 1;				/**< 互斥锁*/
} s_queue;
```


# 程序结构