---
name: Frog Jump Verilog Game
tools: [Verilog, Logic Design, FPGA]
image: basys3.avif
description: A simple hardware-implemented game where a frog jumps to avoid obstacles, built for the Basys3 FPGA board using Verilog and VGA output.
---

# Frog Jump

The **Frog Jump Game** is a straightforward yet engaging project implemented entirely in hardware using Verilog. The game features a small sprite (representing the frog) that must jump to avoid obstacles moving across the screen. It runs on the Basys3 FPGA board, leveraging its VGA output for graphics display.  

This project demonstrates the use of low-level logic design and Verilog to create a functional game without any software processing. The entire gameplay mechanics, obstacle generation, and sprite movement are achieved through combinational and sequential logic implemented in the FPGA.

## Features
- **Platform**: Designed specifically for the Basys3 FPGA board.  
- **Graphics**: Uses VGA output to render the game on an external display.  
- **Logic Design**: Implements game mechanics purely in hardware using Verilog.  
- **Interactive Gameplay**: Players control the frog’s jumps to avoid obstacles.  

## Code  
> **Note:** Only the top-level module is shown here. The complete project, including all modules and design files, can be found on GitHub:  
> [GitHub Repository: Frog Jump Game](https://github.com/bpaoli/Frog-Jump-Game.git)

```verilog
`timescale 1ns / 1ps
//////////////////////////////////////////////////////////////////////////////////
// Company: 
// Engineer: 
// 
// Create Date: 05/22/2022 05:28:56 PM
// Design Name: 
// Module Name: topLevel
// Project Name: 
// Target Devices: 
// Tool Versions: 
// Description: 
// 
// Dependencies: 
// 
// Revision:
// Revision 0.01 - File Created
// Additional Comments:
// 
//////////////////////////////////////////////////////////////////////////////////


module topLevel(
    input btnU,
    input btnD,
    input btnR,
    input btnC,
    input clkin,
    //input btnL,
    //input [15:0]sw,
    output hsync,
    output vsync,
    output [3:0] vgaRed,
    output [3:0] vgaGreen,
    output [3:0] vgaBlue,
    output dp,
    output [6:0] seg,
    output [3:0] an,
    output [15:0] led
    );
    wire enable, blue, frogger, frame,flashsig,  qsec, flash, reset, up, down, clk, digsel,resetscreen,scoreStart, scoreReset, timeStart, timeReset, dead,start, twoSecs, shutter, onesec;
    wire [9:0] x;
    wire [9:0] y;
    assign enable = 1'b1;
    QSEC clock (.Up(frame), .clk(clk), .reset(reset), .qsec(qsec));
    lab7_clks not_so_slow (.clkin(clkin), .greset(btnR), .clk(clk), .digsel(digsel));
    FDRE #(.INIT(1'b0) ) up_sync (.C(clk), .R(reset), .CE(enable), .D(btnU), .Q(up));
    FDRE #(.INIT(1'b0) ) down_sync (.C(clk), .R(reset), .CE(enable), .D(btnD), .Q(down));
    FDRE #(.INIT(1'b0) ) reset_sync (.C(clk), .R(reset), .CE(enable), .D(btnR), .Q(reset));
    FDRE #(.INIT(1'b0) ) start_sync (.C(clk), .R(reset), .CE(enable), .D(btnC), .Q(start));
    //current x and y
    coordinateGenerator coords (.pixelClock(clk), .reset(reset), .x(x[9:0]), .y(y[9:0]), .vsync(vsync), .hsync(hsync), .frame(frame));
    
    // plant logic
    wire [2:0]planty;
    wire [9:0] startxplant0;
    wire [9:0] startxplant1;
    wire [9:0] startxplant2;
    wire [9:0] startyplant0;
    wire [9:0] startyplant1;
    wire [9:0] startyplant2;
    wire LD, count0200, count1200, count2200;
    wire [9:0] plantleftx0;
    wire [9:0] plantleftx1;
    wire [9:0] plantleftx2;
    wire [9:0] planttopy0;
    wire [9:0] planttopy1;
    wire [9:0] planttopy2;
    wire [9:0]plantbottomy0;
    wire [9:0]plantbottomy1;
    wire [9:0]plantbottomy2;
    wire [2:0]count0;
    wire green = planty[0] | planty[1] | planty[2];
    wire [3:0] stater;
    wire [2:0] counthigh;
    plantRandomizer yrandom (.clk(clk), .count0(count0[2:0]), .reset(reset), .y0(startyplant0[9:0]), .y1(startyplant1[9:0]), .y2(startyplant2[9:0]));
    plantCounter plantcount0(.frame(frame), .btnC(start), .reset(reset), .death(dead), .start(scoreStart), .initx(10'd799), .clk(clk), .x(startxplant0[9:0]), .counthigh(counthigh[0]), .countlow(count0[0])); //plant at far right
    plantCounter plantcount1(.frame(frame), .btnC(start), .reset(reset), .death(dead), .start(scoreStart), .initx(10'd559), .clk(clk), .x(startxplant1[9:0]), .counthigh(counthigh[1]), .countlow(count0[1]));
    plantCounter plantcount2(.frame(frame), .btnC(start), .reset(reset), .death(dead), .start(scoreStart), .initx(10'd319), .clk(clk), .x(startxplant2[9:0]), .counthigh(counthigh[2]), .countlow(count0[2])); 
    plants plant0 (.x(x[9:0]), .y(y[9:0]), .startx(startxplant0[9:0]), .starty(startyplant0[9:0]), .leftx(plantleftx0[9:0]), .topy(planttopy0[9:0]), .bottomy(plantbottomy0[9:0]), .plant(planty[0]));
    plants plant1 (.x(x[9:0]), .y(y[9:0]), .startx(startxplant1[9:0]), .starty(startyplant1[9:0]), .leftx(plantleftx1[9:0]), .topy(planttopy1[9:0]), .bottomy(plantbottomy1[9:0]), .plant(planty[1]));
    plants plant2 (.x(x[9:0]), .y(y[9:0]), .startx(startxplant2[9:0]), .starty(startyplant2[9:0]), .leftx(plantleftx2[9:0]), .topy(planttopy2[9:0]), .bottomy(plantbottomy2[9:0]), .plant(planty[2]));

    
    
    //water logic
    water waterr (.x(x[9:0]), .y(y[9:0]), .blue(blue));
    
    //frog logic
    wire [9:0]starty;
    wire [9:0] froglowerx;
    wire [9:0] frogupperx;
    wire [9:0] froglowery;
    wire [9:0] froguppery;
    wire [5:0] p;

    frogJump jumpcontrol(.frame(frame), .reset(reset), .btnC(start),  .clk(clk), .dive(down), .jump(up), .death(dead), .starty(starty[9:0]), .Init(p[5]), .Dive(p[4]), .Diveup(p[3]), .Jump(p[2]), .Jumpdown(p[1]), .Death(p[0]));
    frog froggy (.x(x[9:0]), .y(y[9:0]), .startx(10'd128), .starty(starty[9:0]), .flash(shutter), .color(frogger), .lowerx(froglowerx[9:0]), .lowery(froglowery[9:0]), .upperx(frogupperx[9:0]), .uppery(froguppery[9:0]));
    //vga logic and hit controls
    wire deader;
    wire [3:0] currenttime;
    colorSelector colors (.y(y[9:0]),.x(x[9:0]), .frog(frogger), .blue(blue), .plant(green), .vgaRed(vgaRed[3:0]), .vgaGreen(vgaGreen[3:0]), .vgaBlue(vgaBlue[3:0]));
    assign dead = frogger & green;
    stateMachine states (.start(start), .twoSecs(twoSecs), .dead(dead), .reset(reset), .clk(clk), .blink(flash), .timeStart(timeStart), .resetScreen(resetscreen), .timeReset(timeReset), .scoreStart(scoreStart), .scoreReset(scoreReset), .LD(LD), .deader(deader));
    Timer timecount (.qsec(qsec), .enable(timeStart | deader), .reset(reset|resetscreen), .clk(clk), .out(currenttime[3:0]), .twoSecs(twoSecs), .onesec(onesec));
    //sevenseg logic
    wire [3:0] sel;
    wire [15:0]N;
    wire [3:0]H;
    scoreCounter score (.qsec((counthigh[0] | counthigh[1] | counthigh[2]) & frame), .enable(scoreStart), .reset(reset | scoreReset), .clk(clk), .out(N[15:0]));
    hex7seg segs (.n(H[3:0]), .seg(seg[6:0]));
    ringCounter rings (.Advance(digsel), .clk(clk), .reset(reset), .Q(sel[3:0]));
    selector select (.sel(sel[3:0]), .N(N[15:0]), .H(H[3:0]));
    FDRE #(.INIT(1'b0) ) flasher (.C(clk), .R(reset), .CE(onesec & flash), .D(~shutter), .Q(shutter));
    assign dp = 1'b1;
    assign led[0] = flash;
    assign led[1] = deader;
    assign led[2] = shutter;
    //assign led[3] = timestart;
    assign an[3] = (flash & shutter & deader) | ~(sel[3]);
    assign an[2] = (flash & shutter & deader) | ~(sel[2]);
    assign an[1] = (flash & shutter & deader) | ~(sel[1]);
    assign an[0] = (flash & shutter & deader) | ~(sel[0]);
```