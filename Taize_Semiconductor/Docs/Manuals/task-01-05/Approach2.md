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

### Observation on `./Approach2_data1.txt`
- There is "Outlier Test Cases" that causes dramatic difference from the trends.
- Each kernel shows similar trends.
- Kernels can be both postively and negatively impacted based on the test cases.

#### 1! Still, cannot see the patterns
#### 2! PROBLEM: Even though branch instructions are increased, performance can be improved