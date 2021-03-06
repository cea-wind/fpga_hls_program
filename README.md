# 序

我是在看GPU并行程序设计的相关书籍和资料的时候产生的写书的念头。以CUDA为代表的并行程序设计取得了巨大的成功，但FPGA也有其独特的魅力。本书的很多内容都受到了GPU编程的影响，包括编程模型和硬件系统，但FPGA并行程序设计的思想并不受此限制。

这本书计划主要通过一些例子来阐述一些基本的设计思想，尽管也提供了一些针对FPGA硬件资源和HLS编程的一些说明，但还是假定读者具有相关的背景知识后再来阅读本书。当然，本书的例子足够的简单，可以确保基本不需要任何其他的背景知识。

本书的代码都是用HLS编写的，相对Verilog HDL或者VHDL，HLS的代码可读性更好，编写更为简单。当然，HLS代码最终会编译成硬件代码，因此整个设计过程中依旧需时刻谨记代码在描述硬件的行为。本书在编写过程中还受到了[Ryan Kastner](https://www.semanticscholar.org/author/Ryan-Kastner/1749986)的《Parallel Programming for FPGAs》的影响，这是一本开源的书籍，很容易在网上找到。

写作的过程让我得到了很多的思考，也希望这本书能让你受益。

这本书还处在刚刚开始编写的状态,任何和书籍相关的建议和意见我都会一一阅读参考。计划完成的章节和进度包括

| 章节 | 内容 | 进度 | 计划 |
| :--- | :--- | :--- | :--- |
| 引言 |  | 初稿 | - |
| FPGA硬件概述 | LUT/SRAM等资源简述 | - | 2020/12 |
| HLS编程模型 |  | - | 2020/12 |
| 并行函数设计 | 向量加法/累加/前缀扫描/直方图统计 | 70% | 2020/11 |
| 图像处理方法 | Filter/Image Single Processing | - | Pending |
| 矩阵乘法 | 矩阵乘法和神经网络加速器 | 10% | 2020/11 |

