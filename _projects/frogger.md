---
name: Frog Jump Verilog Game
tools: [Verilog, Logic Design, FPGA]
image: https://www.amd.com/content/dam/amd/en/images/products/boards/2410750-artix-7-xc7a35t-board-product.jpg
description: Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.
---
# Frog Jump

This is a very simple frog jump game where a small sprite has to avoid objects floating across the screen. The project is designed for the Basys3 board over the VGA connection. The entire game is implemented in hardware on the FPGA using low-level logic.

## Code
> **Note:** Only the top-level module is shown here. The codebase for the entire project can be found at:  
> [https://github.com/bpaoli/Frog-Jump-Game.git](https://github.com/bpaoli/Frog-Jump-Game.git)


```systemverilog
module fifo_cdc_1r1w
 #(parameter [31:0] width_p = 32
  ,parameter [31:0] lg_depth_p = 8
  )
  (input [0:0] p_clk_i
  ,input [0:0] p_reset_i
  ,input [width_p - 1:0] p_data_i
  ,input [0:0] p_valid_i
  ,output [0:0] p_ready_o 

  // Use reset to connect to respective producer/consumer sides, but
  // do not cross the clock boundary. It is safe to assume that is
  // handled elsewhere
  ,input [0:0] c_clk_i
  ,input [0:0] c_reset_i
  ,output [0:0] c_valid_o 
  ,output [width_p - 1:0] c_data_o 
  ,input [0:0] c_ready_i
  );
   
  wire [$clog2(lg_depth_p) - 1:0]wrptr_w;
  wire[$clog2(lg_depth_p) - 1:0]rdptr_w;
  wire[$clog2(lg_depth_p) - 1:0]ingrey_w;
  wire[$clog2(lg_depth_p) - 1:0]outgrey_w;
  logic [0:0]c_valid_l;
  logic [0:0] p_ready_l;
  assign c_valid_o = c_valid_l;
  assign p_ready_o = p_ready_l;

  // Write your code here
  ram_1r1w_async
  #(.width_p(width_p)
  ,.depth_p(lg_depth_p))
  fifo
  (.clk_i(p_clk_i)
  ,.reset_i(p_reset_i)

  ,.wr_valid_i(p_valid_i&p_ready_o)
  ,.wr_data_i(p_data_i)
  ,.wr_addr_i(wrptr_w)

  ,.rd_addr_i(rdptr_w)
  ,.rd_data_o(c_data_o)
  );
  
  graycounter//write counter
  #(.width_p($clog2(lg_depth_p)))
  incount
   (.clk_i(p_clk_i)
   ,.reset_i(p_reset_i)
   ,.up_i(p_valid_i & p_ready_o)
   ,.gray_o(ingrey_w));

  graytobin
  #(.width_p($clog2(lg_depth_p)))
  readyoutandptr
  (.gray_i(ingrey_w)
  ,.binary_o(wrptr_w));

  graytobin
  #(.width_p($clog2(lg_depth_p)))
  validout
  (.gray_i(ingrey_w)
  ,.binary_o(validdelay1_w));

  wire [$clog2(lg_depth_p) - 1:0] validdelay1_w;
  logic[$clog2(lg_depth_p) - 1:0] validflop1_l;
  logic [$clog2(lg_depth_p) - 1:0] validflop2_l;

always_ff @(posedge c_clk_i)begin//delay flip flop for the valid out signal
  if(c_reset_i)begin
    validflop1_l <= 0;
  end else begin
    validflop1_l <= validdelay1_w;
  end
end

always_ff @(posedge c_clk_i)begin//delay flip flop for the valid out signal
  if(c_reset_i)begin
    validflop2_l <= 0;
  end else begin
    validflop2_l <= validflop1_l;
  end
end

always_comb begin
  c_valid_l = (validflop2_l !== rdptr_w);
end

  graycounter // read counter
  #(.width_p($clog2(lg_depth_p)))
  outcount
   (.clk_i(c_clk_i)
   ,.reset_i(c_reset_i)
   ,.up_i(c_valid_o & c_ready_i)
   ,.gray_o(outgrey_w));

  graytobin
  #(.width_p($clog2(lg_depth_p)))
  ValidOutAndptr
  (.gray_i(outgrey_w)
  ,.binary_o(rdptr_w));

  wire [$clog2(lg_depth_p) - 1:0] readydelay1_w;
  logic[$clog2(lg_depth_p) - 1:0] readyflop1_l;
  logic [$clog2(lg_depth_p) - 1:0] readyflop2_l;

  graytobin
  #(.width_p($clog2(lg_depth_p)))
  readyout
  (.gray_i(outgrey_w)
  ,.binary_o(readydelay1_w));

  always_ff @(posedge p_clk_i)begin //delay flip flop for the ready out signal
  if(p_reset_i)begin
    readyflop1_l <= 0;
  end else begin
    readyflop1_l <= readydelay1_w;
  end
end

always_ff @(posedge p_clk_i)begin //delay flip flop for the ready out signal
  if(p_reset_i)begin
    readyflop2_l <= 0;
  end else begin
    readyflop2_l <= readyflop1_l;
  end
end
  
always_comb begin
  p_ready_l = (readyflop2_l !== (wrptr_w+1'b1));
end
endmodule
```