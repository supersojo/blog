---
title: NAND Flash接口介绍
date: 2020-01-10 21:10:10
tags:
- NAND Flash
- ONFI
- Toggle Nand
categories:
- NAND
---
## NAND Flash接口介绍

Nand Flash以廉价，高密度得到了广泛应用，这里介绍下Nand相关接口。目前
Nand Flash有两类接口，一类是三星和东芝主导的Toggle NAND接口，一类是其它
Flash厂商包括Cypress主导的ONFI接口。

1. traditional NAND interface
   15 pins with 8 data bus and other control signals
	```
	DQ[7:0] Bidirectional, 8 bits data bus
	CE# input 片选信号
	ALE input
	CLE input
	RE# input
	WE# input
	WP# input
	RY/BY# output 通知控制器操作完成或者空闲
	```
	没有时钟信号，该接口是异步接口
	最高速率达40Mbps
	
2. ONFI NAND v1.0 interface
	add addtional pins for more functions
	```
	IO[7:0]  Bidirectional, 8 bits data bus(Mandatory)
	IO[15:8] Bidirectional, upper 8 bits data bus(Optional)
	IO2[7:0] Bidirectional, second 8 bits data bus(Optional) 
	CE#[3:0] input 支持4个片选
	ALE[1:0] input
	CLE[1:0] input
	RE#[1:0] input
	WE#[1:0] input
	WP#[1:0] input
	RY/BY#[3:0] output
	```
	支持两个8位数据接口或者一个16位数据接口
	最高速率达50Mbps
	
3. Toggle NAND 1.0(Asynchronous DDR NAND Interface)
	performance and throughout
	Added DQS signals
	no clock
	DQS在上升或者下降沿进行数据传输，类似DDR的方式，提高数据传输率。
    最高速率达133Mbps
	
4. ONFI NAND v2.0 interface
	增加时钟信号，支持异步接口保持向后兼容，也可以使用时钟信号支持同步接口
	最高速率达133Mbps
	
5. Toggle NAND 2.0
   采用差分信号和ODT提高数据传输率和信号质量
   最高速率达400Mbps
   
6. ONFINAND3.x和4.x接口
   采用差分信号和ODT提高数据传输率和信号质量
   最高速率达400Mbps
   
7. SPI Nand Flash接口
   低功耗，接口简单
   
## 参考
1. [reference](https://www.embedded.com/flash-101-the-nand-flash-electrical-interface/)