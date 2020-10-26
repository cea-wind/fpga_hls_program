# 附录C：分块矩阵乘法代码详解

### 3.1 并行乘法单元

当完成分块后，最后的计算核心是一组并行的乘法计算单元。这一组计算单元可以完成 $$S_A\times S_B$$ 的矩阵乘法，不失一般性，假设矩阵的大小分别为`PARAL_M*PARAL_K`和`PARAL_M*PARAL_N` ，

```c
PARAL_C_DT MicroMulCore(PARAL_A_DT a_sub_block, PARAL_B_DT b_sub_block) {
  PARAL_C_DT c_sub_block;
  for (int ii = 0; ii < PARAL_M; ii++) {
#pragma HLS UNROLL
    for (int jj = 0; jj < PARAL_N; jj++) {
#pragma HLS UNROLL
      int tmp = 0;
      for (int kk = 0; kk < PARAL_K; kk++) {
#pragma HLS UNROLL
        tmp +=
            a_sub_block.v[ii * PARAL_K + kk] * b_sub_block.v[kk * PARAL_N + jj];
      }
      c_sub_block.v[ii * PARAL_N + jj] = tmp;
    }
  }
  return c_sub_block;
}
```

### 3.2 累加控制单元

```c
void InitCache(PARAL_C_DT cache_reg[4], int cache_idx_i[4],
               int cache_idx_j[4]) {
#pragma HLS INLINE
#pragma HLS DATA_PACK variable = cache_reg
#pragma HLS ARRAY_PARTITION variable = cache_reg complete dim = 1
#pragma HLS ARRAY_PARTITION variable = cache_idx_i complete dim = 1
#pragma HLS ARRAY_PARTITION variable = cache_idx_j complete dim = 1
  for (int i = 0; i < 4; i++) {
#pragma HLS UNROLL
    cache_idx_i[i] = 0;
    cache_idx_j[i] = 0;
  }
}
```



```c
PARAL_C_DT ReadCachedRam(int accum_c[BLOCK_C_HEIGHT][BLOCK_C_WIDTH],
                         PARAL_C_DT cache_reg[4], int cache_idx_i[4],
                         int cache_idx_j[4], int i, int j) {
#pragma HLS INLINE
  PARAL_C_DT rdata;
  for (int t = 0; t < 4; t++) {
#pragma HLS UNROLL
    if (cache_idx_i[t] == i && cache_idx_j[t] == j) {
      rdata = cache_reg[t];
      return rdata;
    }
  }
  for (int r = 0; r < PARAL_M * PARAL_N; r++) {
#pragma HLS UNROLL
    rdata.v[r] = accum_c[i*PARAL_M + r / PARAL_N][j*PARAL_N + r % PARAL_N];
  }
  return rdata;
}
```



```c
void WriteCachedRam(int accum_c[BLOCK_C_HEIGHT][BLOCK_C_WIDTH],
                    PARAL_C_DT cache_reg[4], int cache_idx_i[4],
                    int cache_idx_j[4], int i, int j, PARAL_C_DT wdata) {
#pragma HLS INLINE
  for (int r = 3; r > 0; r--) {
#pragma HLS UNROLL
    cache_reg[r] = cache_reg[r - 1];
    cache_idx_i[r] = cache_idx_i[r - 1];
    cache_idx_j[r] = cache_idx_j[r - 1];
  }
  cache_reg[0] = wdata;
  cache_idx_i[0] = i;
  cache_idx_j[0] = j;
  for (int r = 0; r < PARAL_M * PARAL_N; r++) {
#pragma HLS UNROLL
    accum_c[i*PARAL_M + r / PARAL_N][j*PARAL_N + r % PARAL_N] = wdata.v[r];
  }
}
```

```c
void BlockMulCore(hls::stream<PARAL_A_DT> &matrix_a_strm,
                  hls::stream<PARAL_B_DT> &matrix_b_strm, int a_col,
                  int accum_c[BLOCK_C_HEIGHT][BLOCK_C_WIDTH], bool enable) {
  if (!enable) return;
  PARAL_C_DT cache_reg[4];
  int cache_idx_i[4], cache_idx_j[4];
  InitCache(cache_reg, cache_idx_i, cache_idx_j);

  for (int t = 0; t < a_col; t = t + BLOCK_A_WIDTH) {
    for (int i = 0; i < BLOCK_C_HEIGHT / PARAL_M; i++) {
      for (int j = 0; j < BLOCK_C_WIDTH / PARAL_N; j++) {
        for (int k = 0; k < BLOCK_A_WIDTH / PARAL_K; k++) {
#pragma HLS DEPENDENCE variable = accum_c inter false
#pragma HLS PIPELINE II = 1
          // read
          PARAL_A_DT a_sub_block = matrix_a_strm.read();
          PARAL_B_DT b_sub_block = matrix_b_strm.read();
          PARAL_C_DT update_data =
              ReadCachedRam(accum_c, cache_reg, cache_idx_i, cache_idx_j, i, j);
          PARAL_C_DT axb_sub_block = MicroMulCore(a_sub_block, b_sub_block);
#pragma HLS DATA_PACK variable = update_data
#pragma HLS DATA_PACK variable = axb_sub_block
          // accumlate
          for (int r = 0; r < PARAL_M * PARAL_N; r++) {
#pragma HLS UNROLL
            if (t == 0 && k == 0) {
              update_data.v[r] = axb_sub_block.v[r];
            } else {
              update_data.v[r] = update_data.v[r] + axb_sub_block.v[r];
            }
          }
          // write
          WriteCachedRam(accum_c, cache_reg, cache_idx_i, cache_idx_j, i, j,
                         update_data);
        }
      }
    }
  }
}
```



### 3.3 数据读取单元



```c
void LoadACore(ap_uint<32 * PARAL_K> *a, int col, int row_idx, int col_idx,
               int local_a[BLOCK_A_HEIGHT][BLOCK_A_WIDTH], bool enable) {
#pragma HLS INLINE off
  if (!enable) return;
  for (int i = 0; i < BLOCK_A_HEIGHT; i++) {
    for (int j = 0; j < BLOCK_A_WIDTH / PARAL_K; j++) {
#pragma HLS PIPELINE II = 1
      ap_uint<32 *PARAL_K> tmp =
          a[(row_idx + i) * col / PARAL_K + col_idx / PARAL_K + j];
      for (int k = 0; k < PARAL_K; k++) {
#pragma HLS UNROLL
        local_a[i][j * PARAL_K + k] = tmp.range(k * 32 + 31, k * 32);
      }
    }
  }
}
```

```c
void ProvideACore(int local_a[BLOCK_A_HEIGHT][BLOCK_A_WIDTH],
                  hls::stream<PARAL_A_DT> &matrix_a_strm, bool enable) {
#pragma HLS INLINE off
  if (!enable) return;
  for (int i = 0; i < BLOCK_A_HEIGHT; i = i + PARAL_M) {  // A heigth
    // B width i.e A repeat times
    for (int j = 0; j < BLOCK_B_WIDTH; j = j + PARAL_N) {
      for (int k = 0; k < BLOCK_A_WIDTH; k = k + PARAL_K) {  // A width
#pragma HLS PIPELINE II = 1
        PARAL_A_DT a_sub_block;
        for (int ii = 0; ii < PARAL_M; ii++) {
          for (int kk = 0; kk < PARAL_K; kk++) {
            a_sub_block.v[ii * PARAL_K + kk] = local_a[i + ii][k + kk];
        }
        matrix_a_strm.write(a_sub_block);
      }
    }
  }
}
```



```c

void LoadMatrixA(ap_uint<32 * PARAL_K> *a, int a_row, int a_col, int b_col,
                 hls::stream<PARAL_A_DT> &matrix_a_strm) {
  int local_a[2][BLOCK_A_HEIGHT][BLOCK_A_WIDTH];
#pragma HLS ARRAY_PARTITION variable = local_a complete dim = 1
#pragma HLS ARRAY_PARTITION variable = local_a cyclic factor = \
    local_a_dim2_factor dim = 2
#pragma HLS ARRAY_PARTITION variable = local_a cyclic factor = \
    local_a_dim3_factor dim = 3
  bool pingpang = true;
  for (int i = 0; i < a_row; i = i + BLOCK_A_HEIGHT) {
    for (int r = 0; r < b_col; r = r + BLOCK_B_WIDTH) {
      for (int j = 0; j < a_col; j = j + BLOCK_A_WIDTH) {
        if (pingpang) {
          LoadACore(a, a_col, i, j, local_a[0], true);
          ProvideACore(local_a[1], matrix_a_strm,
                       !(i == 0 && j == 0 && r == 0));
        } else {
          LoadACore(a, a_col, i, j, local_a[1], true);
          ProvideACore(local_a[0], matrix_a_strm,
                       !(i == 0 && j == 0 && r == 0));
        }
        pingpang = !pingpang;
      }
    }
  }
  if (pingpang)
    ProvideACore(local_a[1], matrix_a_strm, true);
  else
    ProvideACore(local_a[0], matrix_a_strm, true);
}
```



### 3.4 数据写回



