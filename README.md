[合集 \- FPGA开发(4\)](https://github.com)[1\.FPGA时序约束基础10\-20](https://github.com/dy-stairmed/p/18487546)[2\.Lattice、Xilinx FPGA reg初始化赋值问题11\-08](https://github.com/dy-stairmed/p/18535369):[楚门加速器](https://chuanggeye.com)[3\.一文讲透 FPGA CDC 多bit跨时钟域同步\-hand\-shanking机制11\-16](https://github.com/dy-stairmed/p/18549428)4\.门控时钟\-无毛刺的时钟切换11\-18收起
# 一、问题


假设存在这样的时钟控制模型：


![](https://ex5xn5y3x9.feishu.cn/space/api/box/stream/download/asynccode/?code=ZjM5YjE0ZjNjMzI1ZjJiYTgzMDRmMmU2ZjM0ZGZhMmRfVUVhUzRVWktkTDE1dUs4dDRUNHdhSkdScXNMTFBOMTdfVG9rZW46VFdOTWI5UHNMb0VYcnZ4ZlRiQWNCYnJCbnpjXzE3MzE5MDgwNjY6MTczMTkxMTY2Nl9WNA)


CLK1、CLK2以及系统时钟的频率与相位均不一致，我们希望在clk\_sel\=1时，输出CLK1,反之输出CLK2，CLK\_SEL可以由系统时钟驱动，也可以由组合逻辑驱动。那么在这种情况下就会出现以下的“毛刺”问题：


![](https://ex5xn5y3x9.feishu.cn/space/api/box/stream/download/asynccode/?code=YWE4MDIyNmFmNTlhN2U2NTY4NTU4NTRmMzk0ODk0YTFfc0FiRWFtNHVKWjRWRFpkanNHUGtXZUlNc2l3eEMwTVRfVG9rZW46WEdmaWJ3Uzc1b0hFQmh4T1ZPQWNIMlJZbnRoXzE3MzE5MDgwNjY6MTczMTkxMTY2Nl9WNA)


可以看到，在CLK\_SEL的交界处，非常容易出现CLK\_OUT时钟出现毛刺的现象，从而影响系统的正常工作。


# 二、解决方法


（以下方法出自B站UP：皮特派）


![](https://ex5xn5y3x9.feishu.cn/space/api/box/stream/download/asynccode/?code=Mjk5YzVkZmZhN2Y1YmUwZGRkNzJlZjg2ZTUwMjRjMzRfYnZNMzUzRnZkbWNta0pMVVMzVElUQ0dtYmU1UW9iN2RfVG9rZW46UXBxVGJMekVzbzJHTVN4THhOT2NnZW5CbmFiXzE3MzE5MDgwNjY6MTczMTkxMTY2Nl9WNA)


具体思路是：1\.两个时钟互斥输出，即B4、B5两个非门作为最后的A2、A4的输入；


2\.S3、S6寄存器采用下降沿驱动对sel进行同步，这样就可以在延迟一拍的情况下做到两个时钟无毛刺的输出。


分析如下：当sel为0时，ali\_2初始值为0，ali\_1为1，A1选通，随后在CLKA的驱动下对alo信号打两拍同步，随后S3在CLKA的下降沿对打拍同步的信号进行采样，这样在下降沿到后一个上升沿之前，由于CLKA为0，CLK\-OUT始终拉低，而后CLKA为高，此时由于上个下降沿已经对ai2\_2进行了同步，CLKA为1，且ai2\_2为1，此时CLK\_OUT输出为CLKA。
本质上是用延迟换准确性。


# 三、代码


根据上述的RTL，代码如下：


1. ## RTL



```
// // =============================================================================
// File Name    : clk_gating_module.v
// Module       : clk_gating_module
// Function     : Burr free clock switching
// Type         : RTL
// Aythor       : Dongyang
// -----------------------------------------------------------------------------
// Update History :
// -----------------------------------------------------------------------------
`timescale 1 ns/1 ns
module  clk_gating_module(
        input            sys_clk    ,
        input            sys_rst_n  ,
  
        input            i_clka     ,
        input            i_clkb     ,
        input            i_clk_sel  ,
        output           o_clk_out  

);

//******************** siganl define ********************
wire         a1i_1        ;
wire         a1i_2        ;
wire         a3i_1        ;
wire         a1o          ;
wire         a3o          ;
wire         a3i_2        ;
wire         a2o          ;
wire         a4o          ;
reg          a4i_2    =  'b0;
reg          a2i_2    =  'b0;
reg  [1:0]   a1o_dly  =  'b0;
reg  [1:0]   a3o_dly  =  'b0;  

//******************** assign  *****************************
assign   a1i_1 =   ~i_clk_sel         ;
assign   a3i_1 =    i_clk_sel         ;
assign   a1i_2 =    ~a4i_2         ;
assign   a3i_2 =    ~a2i_2         ;
assign   a1o   =   a1i_1 & a1i_2 ;
assign   a3o   =   a3i_2 & a3i_1;
assign   a2o   =   i_clka & a2i_2;
assign   a4o   =   a4i_2 & i_clkb;
assign   o_clk_out = a2o | a4o   ;

//**********************always*********************************
// CLKA domain
always @(posedge i_clka) begin
    if(~sys_rst_n) begin
        a1o_dly <= 'b0;
    end
    else begin
        a1o_dly<= {a1o_dly[0],a1o};
    end
end

always @(negedge i_clka) begin
    if(~sys_rst_n) begin
        a2i_2 <= 'b0;
    end
    else  begin
        a2i_2 <= a1o_dly[1];
    end
end

//CLK B domain
always @(posedge i_clkb) begin
    if(~sys_rst_n) begin
        a3o_dly <= 'b0;
    end
    else begin
        a3o_dly<= {a3o_dly[0],a3o};
    end
end

always @(negedge i_clkb) begin
    if(~sys_rst_n) begin
        a4i_2 <= 'b0;
    end
    else  begin
        a4i_2 <= a3o_dly[1];
    end
end

endmodule

```

2. ## TestBench



```
`timescale 1 ns/1 ns
module  tb_clk_gatting();

reg    clka   = 'b0;
reg    clkb   = 'b0;
reg    sys_clk = 'b0;
reg    sys_rst_n = 'b0;
reg    clk_sel  = 'b0 ;

initial begin
clka   = 'b0;
clkb   = 'b0;
sys_clk = 'b0;
sys_rst_n = 'b0;
clk_sel  = 'b0 ;
#6
clkb   = 'b1;
#100
sys_rst_n = 'b1;
#1000
clk_sel  = 1'b1;
#756
clk_sel  = 1'b1;
#1500
clk_sel  = 1'b0;
end

always  # 10   sys_clk = ~sys_clk;    //sys_clk     50M
always  # 50   clka    = ~clka   ;    // CLKA       10M
always  # 30   clkb    = ~clkb   ;     // CLKB     16.6M
clk_gating_module  U_clk_gating_module(
        .sys_clk    (sys_clk),
        .sys_rst_n  (sys_rst_n),
        .i_clka     (clka),
        .i_clkb     (clkb), 
        .i_clk_sel  (clk_sel), 
        .o_clk_out  ()

);

endmodule

```

# 四、仿真波形


![](https://ex5xn5y3x9.feishu.cn/space/api/box/stream/download/asynccode/?code=NThlZjM0ZWZiMjFlZDk3Yjk4MmU4MTRmYWZhODgxNzdfYUphcFIyYWxucWtZWU4ycU1WamlTN0hNTUV6STBISTFfVG9rZW46S20wQ2JsWktKb3FFS054TmNLSmNyZkpBbkloXzE3MzE5MDgwNjY6MTczMTkxMTY2Nl9WNA)


