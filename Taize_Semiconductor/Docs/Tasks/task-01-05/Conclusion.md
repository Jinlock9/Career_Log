# Conclusion

### 1. Machine Code Sinking is likely to HURT pipeline efficiency.  
### 2. Performance (cycle-wise) is indeed biased by test cases.  

A kernel, `unfold`, produced the following test results:  
```sh
TEST[1] unfold(1, 2, 16, 512, 2, 2, 4, 4, true) : OFF: 105231, ON: 93866, DIFF: -10.80%  
TEST[2] unfold(1, 2, 16, 10, 2, 2, 3, 4, true) : OFF: 16889, ON: 16915, DIFF: 0.15%  
TEST[3] unfold(1, 2, 16, 512, 2, 2, 3, 4, true) : OFF: 73743, ON: 74353, DIFF: 0.83%  
```
For test cases 1 and 3, the only difference is the dimension (3 vs. 4).  
In this kernel, execution flows take different control paths depending on the dimension.

Additional tests with `dim = 4` showed performance improvements. 

However, I though there was no reason for cases with `dim = 3` to degrade instead of remaining the same.  
And then performance degradation was observed in a specific loop:  
![img1](.src/img1.png)  
Before machine code sinking, the `load` and `sub` instructions were computed in earlier blocks.  
After sinking, they were moved closer to the `lesser` instruction, which depends on them.  
Compared to the pre-sinking version, the `lesser` instruction had to wait almost 10 additional cycles.

Cases with `dim = 4` also experienced this degradation.  
The performance gain in `dim = 4` cases was due to a different control path at another loop stage, where the gain exceeded the loss.

Therefore, it is concluded that the performance is biased by test cases and machine-sink hurts pipeline efficiency.

### Final Observations  

- Performance is biased by test cases.  
- Among **10 tested kernels**, none exhibited pure performance improvement.  
  - Some test cases showed gains, while others did not.  
- As **matrix sizes increase**, both positive and negative impacts are amplified.  