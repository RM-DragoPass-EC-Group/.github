# 开发规范
## 单位制
距离：m
速度：m*s^(-1)
加速度：m*s^(-2)

角度：rad
角速度：rad*s^(-1)
角加速度：rad*s^(-2)

空间坐标系：机器人运行前方为+x方向，左侧为+y方向，垂直向上为+z方向。

## 数据结构
数据流：uint8_t
整数：int32_t
浮点数：fp32

## Debugging
将遇到的问题，问题出现的原因和解决方案列在github repository的README.md中。

## 版本控制
严格区分Stable和Dev版本代码，使用Git branch区分。

Github每个仓库存放一个开发板运行的程序。

使用[Demo仓库](https://github.com/RM-DragoPass-EC-Group/Demo)练习Git操作。

## 开发环境
强烈建议使用VSCode + Keil Assistant扩展进行开发。使用ARM_Compiler_v5编译。

目前使用ST_Link调试器调试、烧录，未来会尝试使用远程调试器。

## 修改日志
在github repository的README.md中记录每次修改的内容和修改人。

## 组间交流
### 与机械组
电气元件工况（如pitch轴重力补偿）

机械结构参数（如枪口、相机光心和关节位置关系）

走线（预留线材位置和信号链路元件选择）

### 与视觉组
上位机通信协议（发包）

遥控器/键鼠当前设置的键位（以防视觉组调试时误操作）