# Approach 1: Increased Number of Branch Instructions
From the observation, having a negative effect on cycles can be largely due to increased number of branch instruction and jumping. And this is mainly due to *splitting cutting edges*.

Having more branches is inevitable as the price of splitting edges. However, it is still reasonable as newly generated blocks can be skipped while executing, and thus unnecessary computations and register occupation can be avoided.

*Nevertheless, reducing the number of branches is expected to improve the performance.*

#### PROBLEM!: Some kernel shows improvement in performance even though branch instructions are increased.

---

## Suspicion 1: Placing Basic Blocks based on Branch Probability may cause even larger increased branches [X]
From the observation, branch instructions are even more increased after tha **Branch Probability BB Placement**.

### Analysis:
Placing Basic Blocks based on Branch Probability is indeed **BENEFICIAL** even though it introduces more branches.

#### isnan_bf16:
- With Branch Placement 
    - Branches: 
        - `off` to `on`: **14.29%**
        - Machine-Sink=`off`: 49 -> 53 (Uncond: 3 -> 6)   : **8.16%**
        - Machine-Sink=`on` : 56 -> 65 (Uncond: 10 -> 17) : **16.07%**
    - Cycles: **10.08%**
        - Machine-Sink=`off`: 200795 cycles
        - Machine-Sink=`on` : 221045 cycles
- Without Branch Placement 
    - Cycles: **11.26%**
        - Machine-Sink=`off`: 201375 cycles
        - Machine-Sink=`on` : 224054 cycles

#### index_tensor_bf16:
-With Branch Placement 
    - Branches:
        - `off` to `on`: **1.57%**
        - Machine-Sink=`off`: 446 -> 457 (Uncond: 138 -> 145) : **2.47%**
        - Machine-Sink=`on` : 453 -> 472 (Uncond: 145 -> 157) : **4.19%**
    - Cycles: **1.14%**
        - Machine-Sink=`off`: 201206 cycles
        - Machine-Sink=`on` : 203496 cycles
- Without Branch Placement 
    - Cycles: **2.59%**
        - Machine-Sink=`off`: 208490 cycles
        - Machine-Sink=`on` : 213890 cycles

### Conclusion: Having another block placing method (or algorithm) is not a solution for the negative cycles.