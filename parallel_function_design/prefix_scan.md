# 前缀扫描

## 1. 概述

前缀扫描（Prefix Scan），或者被称为前缀和（Prefix Sum）是一种有广泛应用场景的算法。从计算本身而言并不复杂，假设输入为长度为$$n$$的向量 $$\bold{a}$$ ，输出为向量$$\bold{b}$$，计算公式为

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

由于下一次迭代计算依赖上一次迭代的结果，进一步并行化设计的过程会遇到一些困难；和之前的向量加法和向量累加相比，需要重新设计计算的方式和顺序。

## 2. 前缀和的并行算法

一种并行的思路是将向量划分为两段，两段独立计算前缀和。计算完成后将后一段向量计算结果加上前一段的累加和，得到最终的计算结果。计算过程如下所示

$$
b_k = \sum_{i=0}^{k}a_i, k=0,1,...,\frac{n}{2}\\
b_{\frac{n}{2}+k} = \sum_{i=\frac{n}{2}}^{\frac{n}{2}+k}a_{i+\frac{n}{2}}, k=\frac{n}{2} + 1,...,n\\
b_{\frac{n}{2}+k} =b_{\frac{n}{2}+k} +b_{\frac{n}{2}}
$$

按照类似的思路，重复划分输入向量，可以采用向量加法中提到的并行设计方法得到加法的结果，如下图所示。

!\[\]

其中$$a_{ij}=\sum_{k=i}^ja_k$$，如果希望得到最终的前缀和，需要将这些结果组合起来。和只有两段结果的情况不同，这个组合过程变得更为复杂。假设$$b_i = a_i +c_i$$，有

$$
c_{2*i+1}=c_{2*i}+a_{2*i}\\
c_{2*i} = a_{0{i-1}} + \sum_{k=i}^{2*i}a_k
$$

由上式可以看出，一旦求得 $$c_{2*i}$$ ,  可以很容易组合得到$$c_{2*i+1}$$；而求$$c_{2*i}$$的过程可以划分为两个部分，同时利用之前得到的$$a_{ij}=\sum_{k=i}^ja_k$$ 的结果。重复这个过程，可以得到$$c$$的计算过程如下



这种并行求前缀和的算法被称为Blelloch Scan，通过两次扫描高效的完成了计算。

关于前缀和还有另一种求解思路，假设向量$$\bold{d}$$的长度为$$2n$$，且满足

$$
d_i=\left\{\begin{matrix}
0,&&i<n \\ 
a_{i-n},&&i>=n
\end{matrix}\right.
$$

则$$b_k = \sum_{i=k}^{k+n} d_i$$，假设$$b_k^{'} = \sum_{i=k}^{k+\frac{n}{2}} d_i$$，此时有$$b_k = b_k^{'} + b_{k+\frac{n}{2}}^{'}$$。根据这一递推关系，计算过程可表示如下（拓展的0已省略）



这种并行求和的算法被称为 Hillis and Steele Scan，两种并行算法各有其优劣。



## 3. 并行算法的实现

参考上述Hillis and Steele Scan并行算法，可以实现16个输入并行的前缀和算法，重复调用这一计算单元，可以完成任意长度输入的前缀和计算。

!\[\]

实际上在计算16输入并行的前缀和的过程中，Hillis and Steele Scan的硬件实现可以设计为流水进行的，对应的代码实现如下所示。

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

本节探讨了前缀扫描算法的并行实现方式。通过将问题分解，给出了Blelloch Scan和Hillis and Steele Scan两种并行算法，并实际实现了基于Hillis and Steele Scan的HLS代码。Blelloch Scan和Hillis and Steele Scan适应不同的应用场景，感兴趣的通过



