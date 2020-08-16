# 向量加法

## 1. 概述

向量加法是一种计算简单且容易被并行化的计算，其计算可以表示为

$$c[i] = a[i] + b[i], \quad i=0,1,2,...$$

上述计算的代码实现如下。

```c
void vector_add_seq(
    int *a, int *b, int num,
    int *c
) {
    for(int i=0;i<num;i++){
        c[i] = a[i] + b[i];
    }
}
```

其中向量 $$\bold{a,b,c}$$ 均存储再内存中。上述代码在执行过程中，读取一个$$a[i]$$和$$b[i]$$，计算加法后将结果写回到$$c[i]$$中，重复执行这一过程直到计算完成。

若不进行任何修改，直接将这段代码映射到FPGA上，其硬件的处理流程如下图所示。

![&#x4E32;&#x884C;&#x6267;&#x884C;](https://raw.githubusercontent.com/cea-wind/blogs_pictures/master/img20200816021803.png)

从设计上看，这样一种执行方式是极其低效的，程序存在极大的优化空间。

## 2. 流水线并行设计

在向量加法的例子中，取数，计算和写回三个过程是可以并行进行处理的。即在计算第$i$个数的加法运算时，可以预取第$i+1$个数，同时写回第$i-1$个计算结果。优化后的处理流程如下图所示。

![&#x6D41;&#x6C34;&#x7EBF;&#x5E76;&#x884C;&#x8BBE;&#x8BA1;](https://raw.githubusercontent.com/cea-wind/blogs_pictures/master/img20200816021928.png)

这样一种并行处理的设计思路叫做“流水线并行”，顾名思义，指各个处理单元像流水线上的作业形式处理数据，进而达到提高计算速率的目的。只需要添加`pragma`对编译过程进行指示，综合工具能够自动分析并处理流水线并行设计，优化后的代码如下

```c
void vector_add_pipe(
    int *a, int *b, int num,
    int *c
) {
    for(int i=0;i<num;i++){
    #pragma HLS PIPELINE II=1
        c[i] = a[i] + b[i];
    }
}
```

## 3. 数据并行设计

虽然采用了流水线并行，一次计算的平均计算时间变短了，但计算过程依旧时逐个进行的。进一步的，可以采用数据并行设计，一个时钟周期处理多个数据以提高处理性能。这一并行方式类似CPU中的SIMD或GPU中的SIMT的方式。数据并行后的代码如下所示。

```c
void vector_add_parallel(
    ap_uint<512> *a, ap_uint<512> *b, int num,
    ap_uint<512> *c
) {
    for(int i=0;i<num;i=i+16){
    #pragma HLS PIPELINE II=1
        ap_uint<512> rega,regb,regc;
        rega = a[i/16];
        regb = b[i/16];
        for(int j=0;j<16;j++){
        #pragma HLS UNROLL
            regc(32*j+31,32*j) = rega(32*j+31,32*j) +regb(32*j+31,32*j);
        }
        c[i/16] = regc;
    }
}
```

代码中首先将接口从`int`修改成了`ap_uint<512>`，通过这一修改，可以同时读取16个数据。同时，计算过程中将循环拆分成了两个，并在内部循环采用了`UNROLL`的`pragma`对循环进行展开。数据并行后处理流出如下所示。

![&#x6570;&#x636E;&#x5E76;&#x884C;&#x8BBE;&#x8BA1;](https://raw.githubusercontent.com/cea-wind/blogs_pictures/master/img20200816022149.png)

## 4. 总结

本节以向量加法的并行化设计过程为例展示了HLS并行程序设计中两个重要思路，流水线并行和数据并行。这两种并行设计可以分别通过HLS中的`LOOP PIPELINE`和`LOOP UNROLL`来完成。

