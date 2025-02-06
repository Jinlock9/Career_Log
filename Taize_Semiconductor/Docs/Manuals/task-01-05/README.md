# Task: Why Machine Code Sinking has Negative Impact on Kernel Cycles
Need to figure out why this "should have positive impact optimization (Machine code sinking)" have a negative impact on the kernel cycles.

- 2025.01.23 - 01.27, 02.05 - 02.06 [Complete]

#### Example:
- `isnan_bf16`

    | Test | `Without` Machine-Sink | `With` Machine-Sink |
    |:----:|:----------------------:|:-------------------:|
    |   1  |          1975          |         2013        |
    |   2  |          1995          |         2037        |
    |   3  |          1980          |         2033        |
    | ...  |          ....          |         ....        |
    |  10  |         152417         |        159805       |
    |  11  |         200795         |        221045       |
    |  12  |         161445         |        179099       |

## Approaches
1. Increased Number of Branch Instructions
2. Test cases can be Biased

#### See `Conclusion.md`