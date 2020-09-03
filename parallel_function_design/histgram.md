# 直方图统计

## 1. 概述

$$
a = b
$$

```c
void histgram_seq(int *a, int num, int *hist) {
  int local_hist[1024];
  for (int i = 0; i < 1023; i++) {
    local_hist[i] = 0;
  }
  for (int i = 0; i < num; i++) {
    int rega = a[i];
    if (rega < 0) rega = 0;
    if (rega > 1023) rega = 1023;
    local_hist[rega]++;
  }
  for (int i = 0; i < 1023; i++) {
    hist[i] = local_hist[i];
  }
}
```

test $$a = b$$ 

## 2. 流水线设计中的数据依赖

如果简单的针对串行代码增加`PIPELINE` ，会发现综合结果并没有达到预期。进一步分析会发现，使得`II>1` 的原因是计算过程中针对数组`local_hist` 的读写操作存在数据依赖关系，这是一个在之前的所有例子中都没有遇到过的问题。

实际上，由于`local_hist` 是一个长度为1024的数组，在硬件实现过程中被映射成了**SRAM**。对于长度为1024的数组而言，如果采用寄存器实现，那么当需要对数组进行随机寻址时，需要进行一次1024选1的操作，这一操作的计算复杂度极高。除此之外，在进行直方图统计的过程中，一次迭代只需要访问数组中的一个元素，并不需要提供1024个寄存器同时访问的能力，而SRAM的硬件结构很适合这种应用。Xilinx的FPGA中的BRAM（Block RAM）是一种很适合存储1024长度向量的SRAM，和寄存器不同，一般而言BRAM需要1个时钟周期才能完成数据读取的过程。此时，迭代过程中的计算示意图如下。



在一次迭代内部，存在着先写后读（Read After Write, RAW）的数据依赖关系。

### 2.1 消除循环内的读写依赖



```c

void histgram_pipe1(int *a, int num, int *hist) {
#pragma HLS INTERFACE m_axi depth = 2048 port = a
#pragma HLS INTERFACE m_axi depth = 2048 port = hist
  int local_hist[1024];
  for (int i = 0; i < 1024; i++) {
#pragma HLS PIPELINE II = 1
    local_hist[i] = 0;
  }
  int acc = 0;
  int old, cur;
  for (int i = 0; i < num; i++) {
#pragma HLS DEPENDENCE variable = local_hist array intra RAW false
#pragma HLS PIPELINE II = 1
    cur = a[i];
    if (cur < 0) cur = 0;
    if (cur > 1023) cur = 1023;
    if (i == 0 || old == cur) {
      acc = acc + 1;
    } else {
      local_hist[old] = acc;
      acc = local_hist[cur] + 1;
    }
    old = cur;
  }
  local_hist[old] = acc;
  for (int i = 0; i < 1024; i++) {
#pragma HLS PIPELINE II = 1
    hist[i] = local_hist[i];
  }
}

```







### 2.2 设计具有缓存的SRAM结构

```c
void cached_array_init(int local_hist[1024], int cache_reg[4],
                       int cache_idx[4]) {
#pragma HLS INLINE
  for (int i = 0; i < 1024; i++) {
#pragma HLS PIPELINE II = 1
    local_hist[i] = 0;
  }
  for (int i = 0; i < 4; i++) {
#pragma HLS UNROLL
    cache_reg[i] = 0;
    cache_idx[i] = 0;
  }
}
int cached_array_read(int local_hist[1024], int cache_reg[4], int cache_idx[4],
                      int idx) {
#pragma HLS INLINE
  for (int i = 0; i < 4; i++) {
#pragma HLS UNROLL
    if (idx == cache_idx[i]) return cache_reg[i];
  }
  return local_hist[idx];
}
void cached_array_write(int local_hist[1024], int cache_reg[4], int cache_idx[4],
                       int idx, int val) {
#pragma HLS INLINE
  for (int i = 3; i > 0; i--) {
#pragma HLS UNROLL
    cache_reg[i] = cache_reg[i - 1];
    cache_idx[i] = cache_idx[i - 1];
  }
  cache_reg[0] = val;
  cache_idx[0] = idx;
  local_hist[idx] = val;
}
void histgram_pipe2(int *a, int num, int *hist) {
#pragma HLS INTERFACE m_axi depth = 2048 port = a
#pragma HLS INTERFACE m_axi depth = 2048 port = hist
  int local_hist[1024];
#pragma HLS RESOURCE variable=local_hist core=RAM_S2P_BRAM latency=4
  int cache_reg[4];
  int cache_idx[4];
  cached_array_init(local_hist, cache_reg, cache_idx);
  int rega;
  for (int i = 0; i < num; i++) {
#pragma HLS DEPENDENCE variable=local_hist inter RAW distance=5 true
#pragma HLS PIPELINE II = 1
    rega = a[i];
    if (rega < 0) rega = 0;
    if (rega > 1023) rega = 1023;
    int update_val =
        cached_array_read(local_hist, cache_reg, cache_idx, rega) + 1;
    cached_array_write(local_hist, cache_reg, cache_idx, rega, update_val);
  }
  for (int i = 0; i < 1024; i++) {
#pragma HLS PIPELINE II = 1
    hist[i] = cached_array_read(local_hist, cache_reg, cache_idx, i);
  }
}
```



## 3. 多计算单元并行设计





