# Approach 3: The benefits of Machine-Code-Sinking and the benefits of "Pipeline and Multiple Instruction Executions" may be conflicting

#### if:
```asm
LABEL1:
r0 = ld [mem:r1]
...

LABEL2:
r2 = add r0 r1
```
#### After machine-code-sinking:
```asm
LABEL1:
...
LABEL2:
r0 = ld [mem:r1]
r2 = add r0 r1
```

Then: `add` needs to stall until expensive `ld` instruction is finished.
But if `r0 = ld [mem:r1]` was not sunk in to LABEL2, then the data can be prepared early.

#### TODO 1: Needs to verify (from pipeline results)

### Question 1:
#### Example:
```asm
LABEL1:
r1 = r1 + r2
r3 = r4 + r5
...

LABEL2:
r0 = ld [mem:r1]
```
Here, we could sink down `r1 = r1 + r2`, but `r3 = r4 + r5` always needs to be executed. Since two scalar instructions can be executed in parallel, sinking or not sinking won't reduce the time needed in `LABEL1`. Therefore, introducing an analysis of the time reduction could provide a better hint. For example, incorporating a machine scheduler might help optimize the decision.
