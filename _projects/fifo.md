---
name: CDC FIFO
tools: [SystemVerilog, Logic Design, FPGA]
image: fpga.webp
color: primary
description: The CDC FIFO project implements a Clock Domain Crossing FIFO module in SystemVerilog to facilitate seamless data transfer between hardware components operating on different clock domains.
---
# CDC FIFO

This project involves designing a **Clock Domain Crossing (CDC) FIFO (First In, First Out)** module in **SystemVerilog**. A key challenge in hardware design is how to manage data transfer between components with separate clock domains. One common solution to this problem is the use of a **FIFO buffer**, which enables smooth communication between components by decoupling their clock cycles.

The FIFO acts as a buffer between the two components, allowing them to transfer data at different rates without issues related to the relative speed of the hardware. The FIFO module ensures data integrity and synchronization while mitigating the risk of data loss or corruption.

## Design Overview

The FIFO is implemented using **Random Access Memory (RAM)**, which serves as the memory medium. It operates with **ready/valid handshake** protocols for communication between the data producer and consumer. Here's how the handshake works:

1. **Producer**: The data producer asserts the **valid signal** when the data in the FIFO is ready for consumption.
2. **Consumer**: The consumer asserts the **ready signal** when it is prepared to accept data.
3. **Clock Domains**: The producer and consumer operate on separate clocks, and these signals are updated on their respective clock edges.

This system allows for **asynchronous data transfer** between components with independent clock domains, making it an efficient solution for inter-component communication.

## FIFO Code
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

## Testbench Code

```systemverilog
`timescale 1ns/1ps
`ifndef WIDTH
 `define WIDTH 24
`endif
`ifndef LOG2_DEPTH
 `define LOG2_DEPTH 3
`endif
module testbench();
   parameter [31:0]  width_p = `WIDTH;
   parameter [31:0]  lg_depth_p = `LOG2_DEPTH;
   parameter [31:0]  timeout_p = 2000;
   parameter [31:0]  num_tests_p = 256;
   logic [0:0]       reset_done = 1'b0;

   // Producer Interface Signals
   logic [width_p - 1:0] p_data_i;
   logic [width_p - 1:0] p_data_n;
   logic                 p_valid_i;
   logic                 p_valid_n;
   wire                  p_ready_o;
   wire                  p_clk_i;
   wire                  p_reset_i;
   int                   p_data_wr_ptr;
   int                   p_data_wr_ptr_prev;
   int                   p_randval;
   int                   p_cycle;

   // Consumer Interface Signals
   logic [width_p - 1:0] c_data_o;
   logic [width_p - 1:0] c_data_n;
   logic                 c_ready_i;
   logic                 c_ready_n;
   wire                  c_valid_o;
   wire                  c_clk_i;
   wire                  c_reset_i;
   int                   c_data_rd_ptr;
   int                   c_data_rd_ptr_prev;
   int                   c_randval;
   int                   c_cycle;

   // Error signals
   wire                  error;
   wire                  error_data;
   wire                  error_insufficient_rd;
   wire                  error_insufficient_wr;
   wire                  error_timeout;
   wire                  error_invalid_c_valid;
   wire                  error_invalid_p_ready;

   // General Signals
   int                   randval;
   logic [width_p-1:0]   generated_data[num_tests_p-1:0];
   
   nonsynth_clock_gen
     #(.cycle_time_p(11))
   producer_cg
     (.clk_o(p_clk_i));

   nonsynth_reset_gen
     #(.num_clocks_p(1)
       ,.reset_cycles_lo_p(1)
       ,.reset_cycles_hi_p(10))
   producer_rg
     (.clk_i(p_clk_i)
      ,.async_reset_o(p_reset_i));

   nonsynth_clock_gen
     #(.cycle_time_p(17))
   consumer_cg
     (.clk_o(c_clk_i));

   nonsynth_reset_gen
     #(.num_clocks_p(1)
       ,.reset_cycles_lo_p(1)
       ,.reset_cycles_hi_p(10))
   consumer_rg
     (.clk_i(c_clk_i)
      ,.async_reset_o(c_reset_i));

   fifo_cdc_1r1w
     #(.width_p(width_p)
      ,.lg_depth_p(lg_depth_p))
   dut
     (.p_clk_i(p_clk_i)
     ,.p_reset_i(p_reset_i)
     ,.p_data_i(p_data_i)
     ,.p_valid_i(p_valid_i)
     ,.p_ready_o(p_ready_o)

     ,.c_clk_i(c_clk_i)
     ,.c_reset_i(c_reset_i)
     ,.c_valid_o(c_valid_o)
     ,.c_data_o(c_data_o)
     ,.c_ready_i(c_ready_i));

   // Global data initialization
   initial begin

`ifdef VERILATOR
      $dumpfile("verilator.fst");
`else
      $dumpfile("iverilog.vcd");
`endif
      $dumpvars;

      $display();
      $display();
      $display("Begin Test:");
      
      // My normal ascii generation website was down...
      $display("  __                   __ ___.                        .__        _____.__  _____                    .___          ____       ____         ");
      $display("_/  |_  ____   _______/  |\\_ |__   ____   ____   ____ |  |__   _/ ____\\__|/ ____\\____      ____   __| _/____     /_   |_____/_   |_  _  __");
      $display("\\   __\\/ __ \\ /  ___/\\   __\\ __ \\_/ __ \\ /    \\_/ ___\\|  |  \\  \\   __\\|  \\   __\\/  _ \\   _/ ___\\ / __ |/ ___\\     |   \\_  __ \\   \\ \\/ \\/ /");
      $display(" |  | \\  ___/ \\___ \\  |  | | \\_\\ \\  ___/|   |  \\  \\___|   Y  \\  |  |  |  ||  | (  <_> )  \\  \\___/ /_/ \\  \\___     |   ||  | \\/   |\\     / ");
      $display(" |__|  \\___  >____  > |__| |___  /\\___  >___|  /\\___  >___|  /  |__|  |__||__|  \\____/____\\___  >____ |\\___  >____|___||__|  |___| \\/\\_/  ");
      $display("           \\/     \\/           \\/     \\/     \\/     \\/     \\/                       /_____/   \\/     \\/    \\/_____/                       ");


      /* verilator lint_off WIDTH */
      // Fill out test data
      for(logic [31:0] i = 0; i < 10; i++) begin
         // Fill the first few with predictable data.
         generated_data[i] = i;
      end

      for(logic [31:0] i = 10; i < num_tests_p; i++) begin
         randval = $random();
         generated_data[i] = randval;
      end
      /* verilator lint_on WIDTH */
   end


   // Producer Initial Block
   initial begin
      p_data_wr_ptr_prev = '0;
      p_data_n = '0;
      p_valid_n = '0;


      @(negedge p_reset_i);
      repeat(2) @(negedge p_clk_i);
      // This would be unsafe in digital hardware, but it's OK in
      // simulation. We want to wait to make sure we're all good.
      while(c_reset_i === 1'b1)
        @(negedge c_clk_i);

      $display();
      $display("Producer out of reset at %t:", $time);

      if (error_invalid_p_ready)
        $finish();
  

      // Test Full
      // Frequent Writes
      while((p_ready_o === 1) && (p_cycle < timeout_p)) begin
         p_randval = $random();
         p_valid_n = ((p_randval % 4) == 0) && p_ready_o && !p_valid_i;
         p_data_n = p_valid_n ? generated_data[p_data_wr_ptr] : '0;
         $display("On Negedge of Producer Cycle %d (RV: %x) -- Phase 0 -- p_valid_n: %b, p_valid_i: %b, p_ready_o: %d, p_data_i: %x",
                  p_cycle, p_randval, p_valid_n, p_valid_i, p_ready_o, p_data_i);
         @(negedge p_clk_i);
      end
      p_data_wr_ptr_prev = p_data_wr_ptr;

      // Test Empty
      // Infrequent Writes
      while(((p_data_wr_ptr_prev + 10) > p_data_wr_ptr) && (p_cycle < timeout_p)) begin
         p_randval = $random();
         p_valid_n = ((p_randval % 50) == 0) && p_ready_o && !p_valid_i;
         p_data_n = p_valid_n ? generated_data[p_data_wr_ptr] : '0;
         $display("On Negedge of Producer Cycle %d (RV: %x) -- Phase 1 -- p_valid_n: %b, p_valid_i: %b, p_ready_o: %d, p_data_i: %x",
                  p_cycle, p_randval, p_valid_n, p_valid_i, p_ready_o, p_data_i);
         @(negedge p_clk_i);
      end
      p_data_wr_ptr_prev = p_data_wr_ptr;

      // Test 50/50
      while((p_data_wr_ptr != num_tests_p) && (p_cycle < timeout_p)) begin
         p_randval = $random();
         p_valid_n = ((p_randval % 2) == 0) && p_ready_o && !p_valid_i;
         p_data_n = p_valid_n ? generated_data[p_data_wr_ptr] : '0;
         $display("On Negedge of Producer Cycle %d (RV: %x) -- Phase 2 -- p_valid_n: %b, p_valid_i: %b, p_ready_o: %d, p_data_i: %x",
                  p_cycle, p_randval, p_valid_n, p_valid_i, p_ready_o, p_data_i);
         @(negedge p_clk_i);
      end

      if(p_data_wr_ptr == 0)
        $finish();
   end

   always_ff @(posedge p_clk_i) begin
      if(p_reset_i) begin
         p_cycle <= 0;
         p_data_wr_ptr <= '0;
      end else begin
         p_cycle <= p_cycle + 1;
         if((p_valid_i === 1) && (p_ready_o === 1))
           p_data_wr_ptr <= p_data_wr_ptr + 1;
      end

      // We are doing posedge "assignment" negedge checking
      p_valid_i <= p_valid_n;
      p_data_i <= p_data_n;
   end

   // Consumer Initial Block
   initial begin
      c_data_rd_ptr = '0;
      c_data_rd_ptr_prev = '0;
      c_ready_n = '0;

      @(negedge c_reset_i);

      repeat(2) @(negedge c_clk_i);
      // This would be unsafe in digital hardware, but it's OK in
      // simulation. We want to wait to make sure we're all good.
      while(p_reset_i === 1'b1)
        @(negedge c_clk_i);

      $display();
      $display("Consumer out of reset at %t:", $time);

      if (error_invalid_c_valid)
        $finish();


      // Wait a bit to fill more.
      repeat(20) @(negedge c_clk_i);

      // Test Full
      // Infrequent Reads
      while((c_cycle < timeout_p) && (p_ready_o == 1)) begin
         c_randval = $random();
         c_ready_n = ((c_randval % 10) == 0) && c_valid_o && !c_ready_i;
         $display("On Negedge of Consumer Cycle %d (RV: %x) -- Phase 0 -- c_ready_n: %b, c_ready_i: %b, c_valid_o: %d, c_data_o: %x",
                  c_cycle, c_randval, c_ready_n, c_ready_i, c_valid_o, c_data_o);
         @(negedge p_clk_i);
      end

      // Test Empty
      // Frequent Reads 
      while((c_valid_o === 1) && (c_cycle < timeout_p)) begin
         c_randval = $random();
         c_ready_n = ((c_randval % 2) == 0) && c_valid_o && !c_ready_i;
         $display("On Negedge of Consumer Cycle %d (RV: %x) -- Phase 1 -- c_ready_n: %b, c_ready_i: %b, c_valid_o: %d, c_data_o: %x",
                  c_cycle, c_randval, c_ready_n, c_ready_i, c_valid_o, c_data_o);
         @(negedge p_clk_i);
      end

      // Finish remainder
      // Test 50/50
      while((c_data_rd_ptr != num_tests_p) && (c_cycle < timeout_p)) begin
         c_randval = $random();
         c_ready_n = ((c_randval % 2) == 0) && c_valid_o && !c_ready_i;
         $display("On Negedge of Consumer Cycle %d (RV: %x) -- Phase 2 -- c_ready_n: %b, c_ready_i: %b, c_valid_o: %d, c_data_o: %x",
                  c_cycle, c_randval, c_ready_n, c_ready_i, c_valid_o, c_data_o);
         @(negedge p_clk_i);
      end
      $finish();
   end


   always_ff @(posedge c_clk_i) begin
      if(c_reset_i) begin
         c_cycle <= 0;
         c_data_rd_ptr <= '0;
      end else begin
         c_cycle <= c_cycle + 1;
         if((c_valid_o === 1) && (c_ready_i === 1))
           c_data_rd_ptr <= c_data_rd_ptr + 1;
      end

      // We are doing posedge "assignment" negedge checking
      c_ready_i <= c_ready_n;
   end

   always_ff @(negedge c_clk_i) begin
      if(error_data) begin
        $display("\033[0;31m Consumer data is not correct on iteration %d: Expected %x. Got %x. \033[0m", c_data_rd_ptr, generated_data[c_data_rd_ptr], c_data_o);
        $finish();
      end
   end

   assign error_data = (c_valid_o === 1) && (c_ready_i == 1) && (generated_data[c_data_rd_ptr] != c_data_o);
   assign error_insufficient_rd = c_data_rd_ptr != num_tests_p;
   assign error_insufficient_wr = p_data_wr_ptr != num_tests_p;
   assign error_timeout = (c_cycle == timeout_p);

   assign error_invalid_p_ready = (p_ready_o !== 0) && (p_ready_o !== 1);
   assign error_invalid_c_valid = (c_valid_o !== 0) && (c_valid_o !== 1);

   assign error = error_invalid_c_valid
                | error_invalid_p_ready
                | error_data
                | error_insufficient_rd
                | error_insufficient_wr
                | error_timeout;


   final begin
      $display("Simulation time is %t", $time);
      if(error) begin
         
         if(error_invalid_p_ready)
           $display("\033[0;31m Producer ready is not 1/0. Got: %x \033[0m", p_ready_o);

         if(error_invalid_c_valid)
           $display("\033[0;31m Consumer valid is not 1/0. Got: %x \033[0m", c_valid_o);

         if(error_timeout)
            $display("\033[0;31m Transfer timed out!         \033[0m");

         if(error_insufficient_rd)
            $display("\033[0;31m Insufficent data read!         \033[0m");

         if(error_insufficient_wr)
            $display("\033[0;31m Insufficent data written!         \033[0m");

         $display("\033[0;31m    ______                    \033[0m");
         $display("\033[0;31m   / ____/_____________  _____\033[0m");
         $display("\033[0;31m  / __/ / ___/ ___/ __ \\/ ___/\033[0m");
         $display("\033[0;31m / /___/ /  / /  / /_/ / /    \033[0m");
         $display("\033[0;31m/_____/_/  /_/   \\____/_/     \033[0m");
         $display();
         $display("Simulation Failed");
      end else begin
         $display("%d data transmitted", c_data_rd_ptr);
         $display("\033[0;32m    ____  ___   __________\033[0m");
         $display("\033[0;32m   / __ \\/   | / ___/ ___/\033[0m");
         $display("\033[0;32m  / /_/ / /| | \\__ \\\__ \ \033[0m");
         $display("\033[0;32m / ____/ ___ |___/ /__/ / \033[0m");
         $display("\033[0;32m/_/   /_/  |_/____/____/  \033[0m");
         $display();
         $display("Simulation Succeeded!");
      end 
   end
endmodule
```