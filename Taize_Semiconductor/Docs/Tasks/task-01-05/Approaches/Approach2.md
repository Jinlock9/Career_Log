# Approach 2: Test Cases can be Biased

## Initial Thoughts:
Machine Code Sinking offers several benefits, primarily in register optimization and computation reduction. However, its impact on performance is *situational* and can vary based on execution conditions.  
#### 1. Instruction Sinking  
   - Reduces register pressure by shortening the lifetime of registers.  
   - Minimizes computations if the target block (where the instruction is sunk) is not executed.  

#### 2. Critical Edge Cutting
   - Eliminates redundant computations if the newly separated blocks are not executed.  

#### 3. Performance Trade-offs
   - **Potential Gains**: If the sunk instruction or separated block is skipped during execution, the total cycle count is significantly reduced.  
   - **Potential Downsides**: If the newly created blocks are executed, additional branch jumps may increase execution overhead.  
   - The effect on the **critical path** is *unpredictable*—it may become shorter or longer than the original path, making performance improvements dependent on runtime conditions.  

Overall, Machine Code Sinking is a **strategic optimization** that can reduce execution time in favorable scenarios but may introduce additional branching costs in others.

## Suspicion: Test Cases can be Biased
*"You said sinking is gambling, so my understanding would be some of the test cases of isnan would benefit but some would not, (I don't really remember) but seems all test cases have its performance going down, **is it because of the test cases are biased?**"*

### Observation on `./data1.txt`
- There is "Outlier Test Cases" that causes dramatic difference from the trends.
- Each kernel shows similar trends.
- Kernels can be both postively and negatively impacted based on the test cases.

#### 1! Still, cannot see the patterns
#### 2! PROBLEM: Even though branch instructions are increased, performance can be improved

### Large Matrix Causes Performance Drop!

```
KERNEL=eye_bf16
TEST[1] eye_bf16(dlc_bf16, 1024, 1) : OFF: 1560, ON: 1566, DIFF: 0.38%
TEST[2] eye_bf16(dlc_bf16, 895, 1) : OFF: 1557, ON: 1563, DIFF: 0.39%
TEST[3] eye_bf16(dlc_bf16, 768, 1) : OFF: 1553, ON: 1559, DIFF: 0.39%
TEST[4] eye_bf16(dlc_bf16, 1153, 1) : OFF: 1560, ON: 1575, DIFF: 0.96%
TEST[5] eye_bf16(dlc_bf16, 1280, 1) : OFF: 1560, ON: 1575, DIFF: 0.96%
TEST[6] eye_bf16(dlc_bf16, 12, 3) : OFF: 1771, ON: 1859, DIFF: 4.97%
TEST[7] eye_bf16(dlc_bf16, 2048, 2048) : OFF: 561496, ON: 667920, DIFF: 18.95%
[Total] OFF: 571057, ON: 677617, DIFF: 18.66%
----------------------------------------------------------------------------------------------------
KERNEL=eye
TEST[1] eye(dlc_fp32, 768, 1) : OFF: 932, ON: 921, DIFF: -1.18%
TEST[2] eye(dlc_fp32, 895, 1) : OFF: 933, ON: 922, DIFF: -1.18%
TEST[3] eye(dlc_fp32, 1024, 1) : OFF: 934, ON: 923, DIFF: -1.18%
TEST[4] eye(dlc_fp32, 1153, 1) : OFF: 957, ON: 946, DIFF: -1.15%
TEST[5] eye(dlc_fp32, 1280, 1) : OFF: 957, ON: 946, DIFF: -1.15%
TEST[6] eye(dlc_fp32, 12, 3) : OFF: 1150, ON: 1225, DIFF: 6.52%
TEST[7] eye(dlc_fp32, 2048, 2048) : OFF: 346370, ON: 432296, DIFF: 24.81%
[Total] OFF: 352233, ON: 438179, DIFF: 24.40%
```