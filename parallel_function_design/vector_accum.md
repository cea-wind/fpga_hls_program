# 向量累加

## 1. 概述

向量累加是一种很常见的计算，其输入是一长度为$$L$$的向量，输出为一标量。根据输入输出的数据进行分类，这一类计算被成为**归约**（Reduction）。向量累加可以用公式表示如下。

$$
b = \sum_{i=0}^{n-1}{a_i}
$$

为了增加一些设计的难度，假设进行累加计算的数据类型是`float`类型，此时上述计算采用代码表示如下。

```c
void vector_accum_seq(float *a, int num, float *b) {
  b[0] = 0;
  for (int i = 0; i < num; i++) {
    b[0] += a[i];
  }
}
```

后文将基于该段代码进行并行化设计。

## 2. 流水线中的数据依赖

根据上一章向量加法的设计，通过增加`PIPELINE`的`Pramga`，可以轻易的实现流水线并行。据此可以尝试添加流水线并行的设计。

```c
void vector_accum_pipe_bad(float *a, int num, float *b) {
  float regb = 0;
  for (int i = 0; i < num; i++) {
#pragma HLS PIPELINE II = 1
    regb += a[i];
  }
  b[0] = regb;
}
```

除了增加`PIPELINE`的`Pramga`，上述代码还增加一个寄存器`regb`用于暂存计算累加结果，最后再将其赋值给`b[0]`。这是一种常见的访存优化方法，利用寄存器降低对全局内存的访问。否则每次计算过程时都需要对全局内存进行访问，会成为整个系统的瓶颈。

但是上述代码在Vivado HLS 中的综合结果并没有达到`II=1` 的预期（采用UltralScale Plus或之前架构的Xilinx FPGA 存在这一问题，但浮点累加并不一定需要多个周期才能完成）。这是因为浮点的累加计算需要多个周期才能完成（具体执行的周期数取决与采用的器件，设计频率以及Vivado HLS的版本）。在这一情况下，向量累加的示意图如下所示。

![&#x6D6E;&#x70B9;&#x7D2F;&#x52A0;&#x793A;&#x610F;&#x56FE;](https://raw.githubusercontent.com/cea-wind/blogs_pictures/master/img20200903010436.png)

从示意图可以看出，在循环迭代的过程中，由于第`i+1` 次迭代需要使用第`i` 次计算的结果，因此只有在第`i` 次计算完成后才能开始第`i+1` 次计算。这实际上是先写后读（Read After Write，RAW）的数据依赖问题。当浮点数的累加需要一个周期以上时`II>1`。但是设计的计算单元——即浮点加法器——实际上每个周期都能接收新的输入并计算得到结果，因此可以通过设计多个寄存器的方式避免RAW的数据依赖，该设计的示意图如下。

![&#x8BBE;&#x8BA1;&#x6846;&#x56FE;](https://raw.githubusercontent.com/cea-wind/blogs_pictures/master/img20200903011244.png)

![&#x6D6E;&#x70B9;&#x7D2F;&#x52A0;&#x7684;&#x6D41;&#x6C34;&#x7EBF;&#x8BBE;&#x8BA1;](https://raw.githubusercontent.com/cea-wind/blogs_pictures/master/img20200903011442.png)

通过复制多份寄存器，轮流将结果累加到不同的寄存器中，可以解决寄存器RAW的数据依赖，提高计算的性能。最后，将不同寄存器中的结果累加，可以得到最终的计算结果，当输入向量$$\bold{a}$$足够长时，后处理的时间可以忽略。对应代码描述如下（和示意图中不同，代码中一共采用了8个寄存器）。

```c
void vector_accum_pipe_good(float *a, int num, float *b) {
  float regb[8] = {0, 0, 0, 0, 0, 0, 0, 0};
  for (int i = 0; i < num; i++) {
#pragma HLS PIPELINE II = 1
    regb[i % 8] += a[i];
  }
  for (int i = 1; i < 8; i++) {
#pragma HLS PIPELINE II = 8
    regb[0] += regb[i];
  }
  b[0] = regb[0];
}
```

这种解决RAW问题的方案和算法本身密切相关。由于一次只会访问一个`regb` 中的元素，实际实现中可能会采用了SRAM或者Register。

## 3. 数据并行和数组分割

和向量加法中的并行设计类似，也可以进一步通过复制计算单元来提高处理性能。这里依旧使用512bit的数据输入，即每个周期可以读取16个单精度浮点数。此时将`regb`定义为二维数组，将内层循环进行展开以并行完成加法计算。

由于默认情况下，`regb`会被综合成为SRAM，在每个周期只能读取一个数据（FPGA中 可以使用双端口SRAM，一次可以读取两个数据），而在并行的加法计算中，每个周期需要读取16个浮点数据，因此需要对数组进行分割，具体设计下图所示。

![&#x6570;&#x7EC4;&#x5206;&#x5272;&#x8BBE;&#x8BA1;](https://raw.githubusercontent.com/cea-wind/blogs_pictures/master/img20200903013618.png)

对应的代码如下所示，通过使用合适数组分割的方法，可以解决了针对SRAM的访问冲突，最终达到数据并行的目的。在本例中，将数组的第一维完成分割成为了16组独立的SRAM，解决了访问的冲突。

```c
void vector_accum_parallel(ap_uint<512> *a, int num, float *b) {
  float regb[16][8];
#pragma HLS ARRAY_PARTITION variable = regb complete dim = 1
  // initial regb
  for (int i = 0; i < 8; i++) {
#pragma HLS PIPELINE II = 1
    for (int j = 0; j < 16; j++) {
#pragma HLS UNROLL
      regb[j][i] = 0.0f;
    }
  }
  int batch_num = (num + 15) / 16;
  // load 16 data in parallel
  for (int i = 0; i < batch_num; i++) {
#pragma HLS PIPELINE II = 1
    ap_uint<512> rega;
    rega = a[i];
    for (int j = 0; j < 16; j++) {
#pragma HLS UNROLL
      int temp = rega(32 * j + 31, 32 * j);
      regb[j][i % 8] += *(float *)&temp;
    }
  }
  // reduce 16 parallel result
  float regc = 0.0f;
  for (int i = 0; i < 8; i++) {
    for (int j = 0; j < 16; j++) {
#pragma HLS PIPELINE II = 8
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

