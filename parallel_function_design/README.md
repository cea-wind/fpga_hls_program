# 并行函数设计

本章将通过几个具体的并行函数设计实例，阐述并行设计中的思想和实现方案。在阅读本章内容之前，最好对FPGA等相关的硬件有一定的了解，同时清楚HLS的一些基本概念。如果没有，可以参考 “FPGA的硬件概述” 和 “HLS的编程模型” 两章内容进行学习。

本章涉及的所有例子，在函数的接口上包含两种类型，包括指针类型和。在本章中，认为所有的并行函数运行在如下图所示的硬件系统中。



本章的几个例子设计到以下几个知识点，如下表所示。

|  | 向量加法 | 向量累加 | 前缀和 | 直方图统计 |
| :--- | :--- | :--- | :--- | :--- |
| Loop Pipeline |  |  |  |  |
| Loop Unroll |  |  |  |  |
|  |  |  |  |  |
| Array Partition |  |  |  |  |
|  |  |  |  |  |
