FPGA摄像头MIPI协议学习历程
===================
***
关于摄像头mipi的结构
---------------------
目前已知的,mipi从输入到最终取到视频流需要经过几个层次。
分别是：
* 应用层
  * 主要是PS端对于协议那一部分的控制
* 协议层
  * 代指很多,主要就是CSI协议相关
* 物理层
  * 物理层面上各个芯片之间的连接

比较经典的摄像头mipi输入,到跨时钟域处理，以及后续的数据处理的<u>拓扑结构</u>从左到右分别是*Sensor*，*D-PHY IP*，*CSI RX IP*，*Video pipe*和*Video buffer*。

其中D-PHY ip处理的是物理传输级的mipi信号,这里暂时不展开,已知的是该信号包含多个差分信号和多条lane，以及一组i2c信号。该ip将mipi信号转换为我之后要重点讨论的CSI信号。

一般的FPGA厂商会直接提供该IP,在这里我先会直接使用它,至于手撕MIPI协议的任务我将放到之后。

***
关于MIPI CSI-2 Receiver Subsystem的接口学习
---
Xilinx提供了一个可以直接连接图像源的子系统,称作`MIPI CSI-2 Receiver Subsystem`。
 `MIPI CSI-2 Receiver Subsystem:MIPI CSI-2接收子系统
 允许您基于MIPI协议快速创建系统。
 它在基于MIPI的图像传感器和图像传感器管道之间进行接口连接。
 内部提供了一个高速物理层设计——D-PHY,允许直接连接到图像源。
 顶层定制参数可以选择构建子系统所需的硬件模块。`

#### 以下是该子系统的框架示意图
![框架示意图](https://github.com/H0risans/camera_mipi/blob/master/frame.png)
---

事实上,给到该子系统的一般信号,即左上角的一众时钟以及rst信号,再就是AXI4-Lite的接口信号，应该是负责对该子系统的参数做一些调整或是控制，可以暂时先放下不管。

重点关注的是左下角与右下角的信号。
可见该子系统将mipi的物理信号最终转化为axi stream信号,这里也是将重点讨论的部分。

***
MIPI D-PHY的理解
---
关于D-PHY Xilinx给出了单独的数据手册,接下来重点学习该IP。简单来说,该ip即将物理层
信号转化为高级协议CSI-2(also can be DSI,which for displaying),方便后续的信号处理。
#### 以下是该IP的框架示意图
![IP示意图](https://github.com/H0risans/camera_mipi/blob/master/dphy.png)

MIPI D-PHY控制器提供了主设备和从设备之间的点对点连接,或者主机和设备符合相关的MIPI标准。
典型的TX配置包括1个时钟通道和1 ~ 4个数据通道,典型的RX配置包括1个时钟通道和1 ~ 8个数据通道。
主机主要是数据的来源,从设备通常是数据的接收端。
D-PHY通道可以配置为单向通道操作,起源于主设备,终止于从设备。


