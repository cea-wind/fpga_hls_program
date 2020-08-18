# 向量累加

## 1. 概述

向量累加是一种很常见的计算，其输入是一长度为$$L$$的向量，输出为一标量。根据输入输出的数据进行分类，这一类计算被成为**归约**（reduction）。向量累加可以用公式表示如下。

$$
b = \sum_{i=0}^{n-1}{a_i}
$$

为了增加一些设计的难度，假设进行累加计算的数据类型是`float`类型，此时上述计算采用代码表示如下。

```c
void vector_accum_seq(
    float *a, int num,
    float *b
) {
    b[0] = 0;
    for(int i=0;i<num;i++){
        b[0] += a[i];
    }
}
```

后文将基于该段代码进行并行化设计。

## 2. 流水线中的数据依赖

根据上一章向量加法的设计，通过增加`PIPELINE`的`Pramga`，可以轻易的实现流水线并行。据此可以尝试添加流水线并行的设计。

```c
void vector_accum_pipe_bad(
    float *a, int num,
    float *b
) {
    float regb = 0;
    for(int i=0;i<num;i++){
    #pragma HLS PIPELINE II=1
        regb += a[i];
    }
    b[0] = regb;
}
```

除了增加`PIPELINE`的`Pramga`，上述代码还增加一个寄存器`regb`用于暂存计算累加结果，最后再将其赋值给`b[0]`。这是一种常见的访存优化方法，利用寄存器降低对全局内存的访问；否则每次计算过程时都需要对全局内存进行访问。这一过程如下图所示。

!\[\]

但是我们很快会发现，在Vivado HLS中的综合结果并没有达到`II=1` 的预期（采用UltralScale Plus或之前架构的FPGA）。这是因为浮点的累加计算需要多个周期才能完成（具体执行的周期数取决与采用的器件，设计频率以及Vivado HLS的版本）。在这一情况下，向量累加的示意图如下所示。

!\[\]

从示意图可以看出，在循环迭代的过程中，由于第`i+1` 次迭代需要使用第`i` 次计算的结果，因此只有在第`i` 次计算完成后才能开始第`i+1` 次计算。这实际上是先写后读（Read After Write，RAW）的数据依赖问题。

```c
void vector_accum_pipe_good(
    float *a, int num,
    float *b
) {
    float regb[8] = {0,0,0,0,0,0,0,0};
    for(int i=0;i<num;i++){
    #pragma HLS PIPELINE II=1
        regb[i%8] += a[i];
    }
    for(int i=1;i<8;i++){
    #pragma HLS PIPELINE II=8
        regb[0] += regb[i];
    }
    b[0] = regb[0];
}
```

上述代码描述的硬件设计框图如下所示。由于一次只会访问一个`regb` 中的元素，实现中采用了RAM进行实现（LUT Based RAM），以节约硬件资源。

!\[\]框图

## 3. 数据并行和数组分割



```c
void vector_accum_parallel(
    ap_uint<512> *a, int num,
	float *b
) {
    float regb[16][8];
    #pragma HLS ARRAY_PARTITION variable=regb complete dim=1
    // initial regb
    for(int i=0;i<8;i++){
    #pragma HLS PIPELINE II=1
        for(int j=0;j<16;j++){
        #pragma HLS UNROLL
            regb[j][i] = 0.0f;
        }
    }
    // load 16 data in parallel
    for(int i=0;i<(num+15)/16;i++){
    #pragma HLS PIPELINE II=1
    	ap_uint<512> rega;
        rega = a[i];
        for(int j=0;j<16;j++){
        #pragma HLS UNROLL
        	int temp = rega(32*j+31,32*j);
            regb[j][i%8] += *(float *)&temp;
        }
    }  
    // reduce 16 parallel result
    float regc = 0.0f;  
    for(int i=0;i<8;i++){
        for(int j=0;j<16;j++){
		#pragma HLS PIPELINE II=8
            regc += regb[j][i];
        }
    }
    b[0] = regc;
}
```



## 4. 浮点数和计算正确性



## 5. 总结

本节给出了向量累加这种归约算法的并行化实现。在流水线设计的过程中，通过分析算法，利用多组寄存器解决了RAW的问题；在数据并行的设计中，通过对数组进行拆分，解决了同时访问一组SRAM造成的访存冲突的问题。

另外，针对并行算法的结果和串行算法并不一致（实际上同一段代码在不同的计算平台上的计算结果也可能有差异）的问题，还给出了浮点数和浮点计算的分析。

