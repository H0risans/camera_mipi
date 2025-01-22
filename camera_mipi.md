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

其中D-PHY ip处理的是物理传输级的mipi信号,这里暂时不展开,已知的是该信号包含多个差分信号和多条lane，以及一组i2c信号(i2c会被单独处理，并不是交给dphy处理)。该ip将mipi信号转换为我之后要重点讨论的CSI信号。

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
![框架示意图](./frame.png)
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
![IP示意图](./dphy.png)

MIPI D-PHY控制器提供了主设备和从设备之间的点对点连接,或者主机和设备符合相关的MIPI标准。
典型的TX配置包括1个时钟通道和1 ~ 4个数据通道,典型的RX配置包括1个时钟通道和1 ~ 8个数据通道。
主机主要是数据的来源,从设备通常是数据的接收端。
D-PHY通道可以配置为单向通道操作,起源于主设备,终止于从设备。

D-PHY可能包含
  * 低功耗发送器—Low-Power Transmitter(LP-TX)
  * 低功耗接收器—Low-Power Receiver(LP-RX)
  * 高速发送器—High-Speed Transmitter(HS-TX)
  * 高速接收器—High-Speed Receiver(HS-RX)
  * 低功耗竞争检测器—Low-Power Contention Detector(LP-CD)

和三个主要的lane种类
  * 单向时钟Lane
    * Master:HS-TX, LP-TX
    * Slave:HS-RX, LP-RX
  * 单向数据Lane
    * Master:HS-TX, LP-TX
    * Slave:HS-RX, LP-RX
  * 双向数据Lane
    * Master, Slave:HS-TX, LP-TX, HS-RX, LP-RX, LP-CD

简单来说,DPHY的数据接收端是来自摄像头的时间差分与数据差分信号,输出则是hs、lp的信号。
该模块目前所有例程都是通过调用平台ip,所以暂时就了解到这。

仅通过以上的认识并不足以理解mipi的信号,接下来通过纯v代码来继续学习。

***
USB_C_Industrial_Camera_FPGA_USB3-master的学习历程
---
补充一点verilog语法知识：
```verilog
reg [31:0] dword;
reg [7:0] byte0;
reg [7:0] byte1;
reg [7:0] byte2;
reg [7:0] byte3;

assign byte0 = dword[0 +: 8];    // Same as dword[7:0]
assign byte1 = dword[8 +: 8];    // Same as dword[15:8]
assign byte2 = dword[16 +: 8];   // Same as dword[23:16]
assign byte3 = dword[24 +: 8];   // Same as dword[31:24]
```

以下是该例程的top节选。
```verilog
module mipi_csi_16_nx(	//reset_in,
	mipi_clk_p_in,
	mipi_clk_n_in,
	mipi_data_p_in,
	mipi_data_n_in,

        pclk_o,  //data output on pos edge , should be latching into receiver on negedge
	data_o,
	fsync_o, //active high 
	lsync_o, //active high
						
	//these pins may or many not be needed depeding on hardware config
	//cam_ctrl_in, //control camera control input from host
	cam_xce_o, //camera interface selection 
	cam_pwr_en_o, //enable camera power 
	cam_reset_o,  //camera reset to camera
	cam_xmaster_o //camera master or slave 
);

parameter MIPI_LANES = 2;			//number of mipi lanes with camera. Only 2 or 4
parameter MIPI_GEAR = 8;			//deserializer gearing ratio. Only 8 or 16 
parameter MIPI_PIXEL_PER_CLOCK = 2; //number of pixels pipeline process in one clock cycle. With 2 Lanes and Gear 8 only 2 or 4 with gear 16 only 4 , With 4 Lanes only 4 or 8 
parameter MAX_PIXEL_WIDTH = 12;   	//max pixel width , 14bit (RAW14) , IMX219 has 10bit while IMX477 has 12bit and IMX294 has 14bit

input mipi_clk_p_in;
input mipi_clk_n_in;
input [MIPI_LANES-1:0]mipi_data_p_in;
input [MIPI_LANES-1:0]mipi_data_n_in;

wire [((MIPI_LANES * MIPI_GEAR) - 1) :0]mipi_data_raw_hw;    
wire [((MIPI_LANES * MIPI_GEAR) - 1) :0]mipi_data_raw;  

csi_dphy  mipi_csi_phy_inst1(
        .sync_clk_i(osc_clk), 
	.sync_rst_i(1'b0), 
	.lmmi_clk_i(1'b0), 
	.lmmi_resetn_i(1'b0), 
	.lmmi_wdata_i(4'b0), 
	.lmmi_wr_rdn_i(1'b0), 
	.lmmi_offset_i(5'b0), 
	.lmmi_request_i(1'b0), 
	.lmmi_ready_o(), 
	.lmmi_rdata_o(), 
	.lmmi_rdata_valid_o(), 
	//.hs_rx_en_i(1'b1), 
	.hs_rx_clk_en_i(1'b1),  //new
	.hs_rx_data_en_i(1'b1), //new
	.hs_data_des_en_i(1'b1), //new
	.hs_rx_data_o(mipi_data_raw_hw), 
	.hs_rx_data_sync_o(sync_pulse), 
	.lp_rx_en_i(1'b1), 
	.lp_rx_data_p_o(lp_rx_data_p), 
	.lp_rx_data_n_o(), 
	.lp_rx_clk_p_o(lp_rx_clk_p), 
	.lp_rx_clk_n_o(), 
	.pll_lock_i(1'b1), 
	.clk_p_io(mipi_clk_p_in), 
	.clk_n_io(mipi_clk_n_in), 
	.data_p_io(mipi_data_p_in), 
	.data_n_io(mipi_data_n_in), 
	.pd_dphy_i(1'b0), 
	.clk_byte_o(mipi_byte_clock), 
	.ready_o()) ;
```
显然相机能提供的信号主要就是差分时钟和差分数据信号,摄像头一般能给到的raw数据有多种,
不同的格式的处理方式也各不一致。

**在本例程中，仅有2位的mipi物理信号，通过dphy整合后，输出的是16位的raw数据。** 目前该信号的具体组成尚不清楚。

dphy输出的hs、lp信号的理解是接下来的关键。
```verilog
assign mipi_data_raw = mipi_data_raw_hw;

line_reset_generator line_reset_generator_ins0(.clk_i(mipi_byte_clock),
	.lp_data_i(lp_rx_data_p[0]),
  .line_reset_o(line_reset));

generate 
genvar i;
	if (SAMPLE_GENERATOR)
	begin
		wire dummy_byte_valid;	//sample generator should never be active unless debugging 
		assign is_byte_valid = {dummy_byte_valid,dummy_byte_valid};			
		sample_generator sample_generator_ins( .framesync_i(frame_sync),
			.clk_i(mipi_byte_clock),
			.reset_i(line_reset),
		        .byte_o(byte_aligned),
			.byte_valid_o(dummy_byte_valid));
	end
	else
	begin
		for (i = 0;i < MIPI_LANES; i = i +1) 
		begin
			mipi_csi_rx_byte_aligner #(.MIPI_GEAR(MIPI_GEAR))mipi_rx_byte_aligner_0(
                        .clk_i(mipi_byte_clock),
			.reset_i(line_reset),
			.byte_i(mipi_data_raw[(MIPI_GEAR * i) +: MIPI_GEAR]),//0~7
			.byte_o( byte_aligned[(MIPI_GEAR * i) +: MIPI_GEAR]),//8~15
			.byte_valid_o(is_byte_valid[i]));
		end
	end
endgenerate 
```
lp_rx_data_p作为lp的数据输出，最终流入到了line_reset_generator模块当中，并未起到数据传输作用，具体原因目前还不明确。
但是可以看到mipi_data_raw_hw数据确实最终进入了数据处理部分。

以下是在该例程中，有效视频流数据的流向：

```
|camera| 
- |csi_dphy| 
- |mipi_csi_rx_byte_aligner|//多通道例化
- |mipi_csi_rx_lane_aligner|
- |mipi_csi_rx_packet_decoder_*b*lane|   
- |mipi_csi_rx_raw_depacker_*b*lane|
- |debayer_filter|//去马赛克，是之后ISP模块的位置
- |rgb_to_yuv| 
- |output_reformatter|//USB3.0输出
- |上位机显示|
```

**DPHY给出的16位raw信号被一分二，分别传入各自的mipi_csi_rx_byte_aligner中，每八位分开处理。**

### mipi_csi_rx_byte_aligner模块的分析
该模块的工作原理与DPHY的输出信号有关。

该模块的核心语句如下：
```verilog
input [(MIPI_GEAR-1):0]byte_i;
output reg [(MIPI_GEAR-1):0]byte_o;

last_byte 	<= byte_i;
last_byte_2     <= last_byte;
last_word 	<= word;
		
byte_o <= last_word[sync_offset +:MIPI_GEAR]; // from offset MIPI_GEAR upwards
		
byte_valid_o <= valid_reg;
		
if (synced & !valid_reg)// also check for valid_reg to be intive, this make sure that once sync is detected no further sync are concidered till next reset
begin
	sync_offset <= offset;
	valid_reg <= 1'h1;
end
```

