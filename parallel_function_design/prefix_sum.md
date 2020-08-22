# 前缀和

## 1. 概述

前缀和（Prefix Sum），或者被称为前缀扫描（Prefix Scan）是一种有广泛应用场景的算法。从计算本身而言并不复杂，假设输入为长度为$$n$$的向量 $$\bold{a}$$ ，输出为向量$$\bold{b}$$，计算公式为

$$
b_k = \sum_{i=0}^k a_i
$$

根据计算公式，可得$$b_{k+1} = b_{k} + a_{k+1}$$，可以用代码表述如下。

```c
void prefix_sum_seq(int *a, int num, int *b) {

  b[0] = a[0];
  for (int i = 1; i < num; i++) {
    b[i] = b[i - 1] + a[i];
  }
}
```

显然，这是一串行算法，下一次迭代计算以来与上一次迭代的结果。和之前的向量加法和向量累加相比，这个算法的并行化并不那么直接。

## 2. 前缀和的并行算法





## 3. 并行算法的实现

和之前的章节一致，很容易实现一个流水线并行的硬件设计。唯一要注意的是需要设计一个寄存器来缓存之前的累加结果，其实现代码如下所示。

```c
void prefix_sum_pipe(int *a, int num, int *b) {
  int regsum = 0;
  for (int i = 0; i < num; i++) {
#pragma HLS PIPELINE II = 1
    regsum = regsum + a[i];
    b[i] = regsum;
  }
}
```

如果需要进一步进行数据并行的设计，参考上述的并行算法，可以实现16个输入并行的前缀和算法，重复调用这一计算单元，可以完成任意长度输入的前缀和计算。

!\[\]

实际上在计算16输入并行的前缀和的过程中，硬件实现也是流水进行的。对应的代码实现如下所示。

```c
void prefix_sum_parallel(ap_uint<512> *a, int num, ap_uint<512> *b) {
  ap_uint<512> rega, regb;
  int regs1[16], regs2[16], regs3[16], regs4[16];
  int regprev = 0;
  int batch_num = (num + 15) / 16;
  for (int i = 0; i < batch_num; i = i + 1) {
#pragma HLS PIPELINE II = 1
    rega = a[i];
    for (int j = 0; j < 16; j++) {
      if (j >= 1)
        regs1[j] = rega(32 * j + 31, 32 * j) + rega(32 * j - 1, 32 * j - 32);
      else
        regs1[j] = rega(31, 0);
    }
    for (int j = 0; j < 16; j++) {
      if (j >= 2)
        regs2[j] = regs1[j] + regs1[j - 2];
      else
        regs2[j] = regs1[j];
    }
    for (int j = 0; j < 16; j++) {
      if (j >= 4)
        regs3[j] = regs2[j] + regs2[j - 4];
      else
        regs3[j] = regs2[j];
    }
    for (int j = 0; j < 16; j++) {
      if (j >= 8)
        regs4[j] = regs3[j] + regs3[j - 8];
      else
        regs4[j] = regs3[j];
    }
    for (int j = 0; j < 16; j++) {
      regb(32 * j + 31, 32 * j) = regs4[j] + regprev;
    }
    regprev = regb(511, 480);
    b[i] = regb;
  }
}
```





## 4. 总结



