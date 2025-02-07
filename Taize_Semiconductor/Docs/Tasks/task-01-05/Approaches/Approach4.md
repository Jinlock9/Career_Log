# Approach 4: Performance can be biased due to the Type of Kernels

### `mean_dim_bf16`
```sql
TEST[1] mean_dim_bf16(1, 1, 5, 2, 6, {0, 0, 1, 0, 1}, false, true) : OFF: 3500, ON: 3434, DIFF: -1.89%
TEST[2] mean_dim_bf16(1, 1, 1, 8, 256, {0, 0, 0, 0, 1}, false, true) : OFF: 2319, ON: 2298, DIFF: -0.91%
TEST[3] mean_dim_bf16(1, 1, 1, 53, 288, {0, 0, 0, 0, 1}, false, true) : OFF: 3645, ON: 3622, DIFF: -0.63%
TEST[4] mean_dim_bf16(1, 1, 1, 53, 4096, {0, 0, 0, 0, 1}, false, true) : OFF: 4917, ON: 4894, DIFF: -0.47%
TEST[5] mean_dim_bf16(1, 7, 1, 2, 1111, {0, 1, 1, 0, 1}, false, true) : OFF: 4446, ON: 4448, DIFF: 0.04%
TEST[6] mean_dim_bf16(2, 2, 2, 262, 263, {0, 1, 1, 0, 1}, false, true) : OFF: 207237, ON: 208211, DIFF: 0.47%
[Total] OFF: 226064, ON: 226907, DIFF: 0.37%
```

### `nansum_intList_bf16`

### `unfold`
```sql
TEST[1] unfold(1, 2, 16, 512, 2, 2, 4, 4, true) : OFF: 105231, ON: 93866, DIFF: -10.80%
TEST[2] unfold(1, 2, 16, 10, 2, 2, 3, 4, true) : OFF: 16889, ON: 16915, DIFF: 0.15%
TEST[3] unfold(1, 2, 16, 512, 2, 2, 3, 4, true) : OFF: 73743, ON: 74353, DIFF: 0.83%
[Total] OFF: 195863, ON: 185134, DIFF: -5.48%
```

### `eye`
```sql
TEST[1] eye(dlc_fp32, 768, 1) : OFF: 932, ON: 921, DIFF: -1.18%
TEST[2] eye(dlc_fp32, 895, 1) : OFF: 933, ON: 922, DIFF: -1.18%
TEST[3] eye(dlc_fp32, 1024, 1) : OFF: 934, ON: 923, DIFF: -1.18%
TEST[4] eye(dlc_fp32, 1153, 1) : OFF: 957, ON: 946, DIFF: -1.15%
TEST[5] eye(dlc_fp32, 1280, 1) : OFF: 957, ON: 946, DIFF: -1.15%
TEST[6] eye(dlc_fp32, 12, 3) : OFF: 1150, ON: 1225, DIFF: 6.52%
TEST[7] eye(dlc_fp32, 2048, 2048) : OFF: 346370, ON: 432296, DIFF: 24.81%
[Total] OFF: 352233, ON: 438179, DIFF: 24.40%
```