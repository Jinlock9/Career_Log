# Machine Code Sink in LLVM

## Machine Code Sink
**Machine Code Sink** (often referred to as **Machine Instruction Sinking**) in LLVM is an optimization technique performed during the backend compilation phase. It involves moving machine instructions to later positions in the instruction stream while preserving program semantics. This optimization is aimed at reducing register pressure, improving scheduling, and enhancing the overall performance of the generated code.

### Key Concepts of Machine Instruction Sinking

1. **Definition**:
   - Sinking moves instructions closer to their point of use, typically below control flow branches or into basic blocks where they are actually needed.

2. **Benefits**:
   - **Reduced Register Pressure**: By delaying the computation of values until they are needed, the compiler can reduce the lifetime of registers, freeing them up for other uses.
   - **Improved Scheduling**: Instructions are positioned closer to their consumers, allowing for better instruction-level parallelism and scheduling opportunities.
   - **Code Size**: Sometimes, sinking can eliminate redundant computations in paths where results are not used.

3. **How It Works**:
   - Identify instructions that are safe to sink. These are typically:
     - Not used in earlier code paths.
     - Free of side effects (e.g., not memory writes or I/O instructions).
     - Not dependent on instructions that are conditionally executed.
   - Move these instructions into successor basic blocks, provided the move doesn't violate dependencies or alter observable behavior.

4. **Example**:

   **Before Sinking**:
   ```assembly
   BB1:
     %1 = expensive_computation
     br label BB2
   BB2:
     use(%1)
   ```

   **After Sinking**:
   ```assembly
   BB1:
     br label BB2
   BB2:
     %1 = expensive_computation
     use(%1)
   ```

   Here, `%1` is computed only when it is actually needed in `BB2`.

---

## Implementation of Machine Code Sink

### Process Block
- `MBB`(Machine Basic Block) [`MI`] <- **Scan instructions from bottom to top**.

```cpp
bool Joined = PerformTrivialForwardCoalescing(MI, &MBB);
if (Joined) {
    MadeChange = true;
    continue;
}

if (SinkInstruction(MI, SawStore, AllSuccessors)) {
    ++NumSunk;
    MadeChange = true;
}
```
#### Explanation
1. `ProcessBlock` Functionality:

    This block of code iterates through the instructions in a Machine Basic Block (MBB).
    The scan occurs from bottom to top, meaning it starts with the last instruction in the block and works upwards.

2. Instruction Sinking Logic:

    - The loop attempts to either:
        1. **Coalesce trivial instructions** using `PerformTrivialForwardCoalescing`, or
        2. **Sink instructions** using `SinkInstruction`.

3. Code Breakdown:

    - `PerformTrivialForwardCoalescing`:

        - This function attempts to simplify the control flow by forward-coalescing instructions when possible.
        - For example, if an instruction’s computation can be directly folded into a subsequent instruction, this optimization occurs.
        - If coalescing is successful (`Joined` is true), the function marks the block as having been modified (`MadeChange = true`) and skips to the next instruction (`continue`).

    - `SinkInstruction`:

        - If trivial coalescing isn’t possible, the code attempts to sink the current instruction (`MI`).
        - The parameters passed include:
            - `MI`: The current Machine Instruction being analyzed.
            - `SawStore`: Tracks whether a memory store has been encountered (affecting sinking decisions).
            - `AllSuccessors`: Tracks the successor basic blocks for dependency checks.
        - If the sinking succeeds:
            - The counter `NumSunk` is incremented.
            - The block is marked as modified (`MadeChange = true`).

4. Outcome:

    The loop continues processing instructions in the block until all possible instructions are sunk or coalesced.

---

### `PerformTrivialForwardCoalescing`
#### Diagram
```sql
+-------------------+
|   ParentBlock     |
|   DefMI: Op Src   |
|   MI: COPY Dst, Src|
+-------------------+
         |
         | Replace all uses of Dst with Src
         v
+-------------------+
|   ParentBlock     |
|   DefMI: Op Src   |
+-------------------+

```

```cpp
if (!TargetRegisterInfo::isVirtualRegister(SrcReg) ||
    !TargetRegisterInfo::isVirtualRegister(DstReg) ||
    MRI->hasOneNonDBGUse(SrcReg)) {
    return false;
}
       
MRI->replaceRegWith(DstReg, SrcReg);
MI.eraseFromParent();
```

This diagram provides an explanation of **Trivial Forward Coalescing** in LLVM, which is part of the **Machine Instruction Optimization** process. It demonstrates how unnecessary `COPY` instructions can be eliminated by directly substituting the registers involved. Here's a detailed breakdown:


#### Explanation

1. **Before Optimization (Up Side):**
   - In the **Machine Basic Block (MBB)**, there are two instructions:
     - **`DefMI`**: This is the instruction that defines the `SrcReg` (source register). For example:
       ```
       DefMI: Op SrcReg, ...
       ```
     - **`MI`**: This is a `COPY` instruction that copies the value from `SrcReg` into `DstReg` (destination register). For example:
       ```
       MI: COPY DstReg, SrcReg
       ```
   - The `COPY` instruction in this case is trivial because `DstReg` can directly use `SrcReg`.

2. **Optimization Process:**
   - The `PerformTrivialForwardCoalescing` function checks if the `COPY` instruction is redundant.
   - Conditions checked:
     - `SrcReg` and `DstReg` must be virtual registers.
     - `SrcReg` must have exactly one non-debug use.
   - If the above conditions are satisfied:
     - The `COPY` instruction is eliminated.
     - All uses of `DstReg` are replaced by `SrcReg`.

   **Code Snippet:**
   ```cpp
   if (!TargetRegisterInfo::isVirtualRegister(SrcReg) ||
       !TargetRegisterInfo::isVirtualRegister(DstReg) ||
       MRI->hasOneNonDBGUse(SrcReg))
       return false;

   MRI->replaceRegWith(DstReg, SrcReg);
   MI.eraseFromParent();
   ```

   - **`MRI->replaceRegWith(DstReg, SrcReg)`**:
     - Updates all uses of `DstReg` in the program to use `SrcReg` instead.
   - **`MI.eraseFromParent()`**:
     - Removes the `COPY` instruction from the parent block (i.e., deletes it).

3. **After Optimization (Down Side):**
   - The `COPY` instruction is removed.
   - The `DstReg` is no longer explicitly defined because all references to it now use `SrcReg` directly.
   - The resulting **MBB** is simplified to only contain:
     ```
     DefMI: Op SrcReg, ...
     ```

---

### `SinkInstruction`
#### Diagram
```sql
+-------------------+
|   ParentBlock     |  (Contains MI)
|       MI          |
+-------------------+
         |
         v
+-------------------+
|   SuccToSinkTo    |  (Chosen successor block)
+-------------------+
         |
         v
+-------------------+
|  PHI Instructions |  (Ensure MI is inserted after these)
+-------------------+
         |
         v
+-------------------+
|  Insert MI here   |  (Non-PHI position)
+-------------------+
  
```
If `ParentBlock` does not dominate `SuccToSinkTo`, or `SuccToSinkTo` is a loop header, try to split the edge between `ParentBlock` and `SuccToSinkTo`.  
Refer to `PostponeSplitCriticalEdge()`.

```cpp
MachineBasicBlock::iterator InsertPos = SuccToSinkTo->begin();
while (InsertPos != SuccToSinkTo->end() && InsertPos->isPHI()) {
    ++InsertPos;
}
performSink(MI, *SuccToSinkTo, InsertPos);
```
This diagram illustrates the **SinkInstruction** process in LLVM. It shows how an instruction (`MI`) in a **ParentBlock** is moved (sunk) into a successor block (`SuccToSinkTo`), as part of the **instruction sinking optimization**. Here's a detailed explanation of the components:

#### Key Concepts of `SinkInstruction`

1. **ParentBlock**:
   - The **Machine Basic Block (MBB)** where the instruction `MI` originally resides.
   - `MI` is an instruction that can potentially be sunk into one or more of its successor blocks.

2. **SuccToSinkTo**:
   - The successor block where the instruction is moved (sunk).
   - Sinking occurs when it's determined that the instruction `MI` is only needed in `SuccToSinkTo` and not in other paths.

3. **PHI Instructions**:
   - If `SuccToSinkTo` contains **PHI instructions**, the sunk instruction (`MI`) must be placed *after* all PHI nodes to maintain proper SSA (Static Single Assignment) form.
   - This is because PHI nodes logically occur at the beginning of a block, and any sunk instructions must follow them.

#### Flow of `SinkInstruction`

1. **Precondition Checks**:
   - Before sinking, LLVM checks:
     - Whether the **ParentBlock dominates** `SuccToSinkTo`.
       - Dominance ensures that `MI` is executed whenever the successor block executes.
     - Whether `SuccToSinkTo` is a loop header.
       - If `SuccToSinkTo` is a loop header or the dominance condition is not met, the edge between the two blocks might need to be **split**.
       - This is handled by a helper function like `PostponeSplitCriticalEdge()`.

2. **Finding the Insert Position**:
   - An iterator (`InsertPos`) is initialized to the start of `SuccToSinkTo`.
   - The iterator moves forward, skipping any **PHI instructions**:
     ```cpp
     MachineBasicBlock::iterator InsertPos = SuccToSinkTo->begin();
     while (InsertPos != SuccToSinkTo->end() && InsertPos->isPHI())
         ++InsertPos;
     ```
   - The first non-PHI position becomes the insertion point for the sunk instruction.

3. **Perform Sinking**:
   - The instruction (`MI`) is sunk into the successor block at the identified insertion point:
     ```cpp
     performSink(MI, *SuccToSinkTo, InsertPos);
     ```
   - This involves:
     - Physically moving `MI` from the ParentBlock to `SuccToSinkTo`.
     - Updating any dependencies or operands affected by the move.

#### Special Case: Splitting Critical Edges
- If the **ParentBlock does not dominate `SuccToSinkTo`**, or if `SuccToSinkTo` is a **loop header**, LLVM might split the edge between the two blocks.
- This creates a new intermediate block where `MI` can be safely moved without violating dominance or control flow properties.
- This process is managed by functions like `PostponeSplitCriticalEdge()`.

---

### `FindSuccToSinkTo`
#### Diagram
```sql
          +----------+
          |   MBB    |  (Parent Block containing MI)
          +----------+
           /    |     \
          v     v      v
   +----------+ +----------+ +----------+
   | Succ #1  | | Succ #2  | | Succ #3  |  (All Successors)
   +----------+ +----------+ +----------+
           |
           v
   +-----------------+
   |  SuccToSinkTo   |  (Dominates all uses of MI)
   +-----------------+
           |
           v
   +-----------------+
   | Use of MI       |  (Dependent on MI)
   +-----------------+
```
1. `SuccToSinkTo` dominates all `MI` use operands.
2. Sort all successors by frequency.
    - Smaller frequency, higher priority.
    - Get the highest priority block that dominates all uses of MI as `SuccToSinkTo`.


This diagram explains the process of selecting the appropriate successor block (`SuccToSinkTo`) during the **instruction sinking** optimization in LLVM. The goal is to identify the best successor block where an instruction (`MI`) can be safely moved to while minimizing execution overhead and preserving correctness.

#### Explanation of the Diagram

1. **`MBB`**:
   - This is the **parent block** containing the instruction `MI` (the instruction being considered for sinking).

2. **All Successors**:
   - The parent block has multiple successor blocks (connected via control flow edges).

3. **`SuccToSinkTo`**:
   - This is the chosen successor block where the instruction `MI` will be sunk.
   - The selection is made based on:
     1. **Dominance**:
        - `SuccToSinkTo` must dominate all uses of `MI` to ensure the instruction is executed before any of its uses.
     2. **Execution Frequency**:
        - Successors are sorted by execution frequency (typically gathered from profiling information). Blocks with **lower frequency** are given higher priority, as sinking to less frequently executed blocks reduces overhead.

4. **Use of `MI`**:
   - The chosen successor block must ensure that all uses of `MI` (operands that depend on its result) are still valid after sinking.

#### Steps to Select `SuccToSinkTo`

1. **Filter Successors**:
   - Check all successor blocks of `MBB`.
   - Only consider successors that **dominate all uses** of `MI`.

2. **Sort by Frequency**:
   - Among the valid successors, sort them by **execution frequency**:
     - Lower frequency = Higher priority.

3. **Choose the Best Block**:
   - Select the block with the highest priority (lowest execution frequency) as `SuccToSinkTo`.

---

### `AllUsesDominatedByBlock`
#### Diagrams
- Case 1: `BreakPHIEdge = true`
```sql
+-----------------+
|    DefMBB       |  (Defining Block)
|    Reg = ...    |
+-----------------+
        |
        v
+-----------------+
|      MBB        |  (Contains PHI instruction)
| phi(Reg, DefMBB, ...)
+-----------------+
        |
   return true
```
- Case 2: `BreakPHIEdge = false`
```sql
+-----------------+
|    DefMBB       |  (Defining Block)
|    Reg = ...    |
+-----------------+
        |
        v
+-----------------+
|   UseBlock      |  (Dominated by DefMBB)
|   X = Reg ...   |
+-----------------+
        |
   return true
```
- Case 3: `LocalUse = true`
```sql
+-----------------+
|   DefMBB        |  (Defining Block == Use Block)
|   Reg = ...     |
|   X = Reg ...   |  (Use within the same block)
+-----------------+
        |
   return false
```

This diagram illustrates the **AllUsesDominatedByBlock** concept, which determines whether all uses of a value defined in one basic block are dominated by another block (or the same block). This is critical for safe instruction sinking or transformations.

#### Explanation

The diagram breaks down into three scenarios, each showing conditions for checking whether all uses of a value are dominated by a block.

#### **Case 1: `BreakPHIEdge = true`**
- **Description**:
  - The value (`Reg`) is defined in `DefMBB` (Definition Block).
  - The value is used in a **PHI instruction** in another block (`MBB`).
  - PHI instructions gather values from predecessors of the block where the PHI resides.
- **Outcome**:
  - The value's use (via `phi(Reg, DefMBB, ...)`) is considered **dominated** because the defining block (`DefMBB`) is a valid predecessor of the block containing the PHI instruction.
- **Return Value**:
  - **`true`**, as all uses are dominated by `DefMBB`.

#### **Case 2: `BreakPHIEdge = false`**
- **Description**:
  - The value (`Reg`) is defined in `DefMBB` and used in `UseBlock` via `X = Reg ...`.
  - The defining block (`DefMBB`) **dominates** the `UseBlock`.
- **Outcome**:
  - Since `DefMBB` dominates `UseBlock`, all paths to the use pass through `DefMBB`, ensuring correctness.
- **Return Value**:
  - **`true`**, as all uses are dominated by `DefMBB`.

#### **Case 3: `LocalUse = true`**
- **Description**:
  - The value (`Reg`) is defined in `DefMBB` and used within the same block (`UseBlock == DefMBB`).
  - The value is immediately used (`X = Reg ...`) after being defined.
- **Outcome**:
  - Even though the use occurs in the same block, it is not explicitly **dominated** by `DefMBB` due to potential dependencies or control flow inconsistencies.
- **Return Value**:
  - **`false`**, as this case doesn't meet the dominance criteria for all paths.

---

### `PostponeSplitCriticalEdge`
#### Diagram
We do not split the edges on the fly. Indeed, this invalidates the dominance information and thus triggers a lot of updates of that information underneath.  
Instead, we postpone all the splits after each iteration of the main loop. That way, the information is at least valid for the lifetime of an iteration.

- Before Splitting
```sql
   +-------+       
   |  bb.1 |       
   | v1024 |       
   +-------+       
      |     \     
      |      \
      v       v
   +-------+  +-------+
   |  bb.2 |  |  bb.3 |    
   |       |  | x =   |
   +-------+  | v1024 |
              +-------+
```
**Critical Edge**: An edge whose source has multiple successors and whose sink has multiple predecessors.

- After Splitting
```sql
   +-------+
   |  bb.1 |  (From: Defines v1024)
   +-------+
      |
      v
   +-----------+
   | new block |  (Splits edge)
   | v1024 =   |
   +-----------+
      |     \
      v      v
   +-------+  +-------+
   |  bb.2 |  |  bb.3 |  (To: Uses v1024)
   +-------+  | x =   |
              | v1024 |
              +-------+

```
It is wrong result. `v1024` is not defined in the path `bb.1 -> bb.2 -> bb.3`.

There is only one path from 'new block' to `bb.3`. So, splitting edges is only valid when `bb.3` dominates all the predecessors besides `bb.1`. That is, 'To block' is dominates all its predecessors besides `From block`.

```cpp
for (PI = ToBB->pred_begin(), E = ToBB->pred_end(); PI != E; ++PI) {
    if (*PI == FromBB) {
        continue;
    }
    if (!DT->dominates(ToBB, *PI)) {
        return false;
    }
} 
```

#### Explanation

This image further elaborates on the **PostponeSplitCriticalEdge** strategy in LLVM, focusing on when and how to safely split critical edges. It discusses dominance relationships and shows an example of improper edge splitting leading to incorrect results.

#### Key Points

1. **Critical Edge**:
   - A **critical edge** is an edge in the control flow graph where:
     - The **source block** has multiple successors.
     - The **destination block** has multiple predecessors.

2. **Postpone Splitting**:
   - Critical edges are not split immediately during transformations to avoid invalidating dominance information, which would require expensive recalculations. Instead:
     - Splits are **postponed until the end** of an optimization loop iteration.

3. **Conditions for Splitting**:
   - Splitting is only valid if the **destination block (`To`) dominates all its predecessors except the **source block (`From`).
   - If this condition is not met, splitting the edge might lead to incorrect results because the required values (like `v1024` in the diagram) may not be defined for all paths.

4. **Code Example**:
   - The provided pseudocode iterates through all predecessors (`pred_begin`) of the destination block (`ToBB`) and checks:
     - If a predecessor is not the source block (`FromBB`).
     - Whether the destination block (`ToBB`) **dominates** that predecessor.
   - If any predecessor violates this condition, splitting is invalid.

#### Incorrect Splitting Scenario

1. **Initial CFG**:
   - Block `bb.1` defines a value `v1024`.
   - Block `bb.1` has two successors: `new block` (introduced during splitting) and `bb.2`.
   - Block `bb.3` uses `v1024`.

2. **Incorrect Path**:
   - After splitting, there is a path `bb.1 -> bb.2 -> bb.3` where `v1024` is **not defined**, leading to incorrect behavior.

3. **Correctness Condition**:
   - Splitting is only valid if `bb.3` dominates all predecessors (except `bb.1`). This ensures `v1024` is properly defined for all paths leading to `bb.3`.

---

### `SplitCriticalEdge`
#### Diagram Representation**
- **Before Splitting**
```sql
   +------+
   |  BB  |  (Source block)
   +------+
      |
      v
   +------+
   | Succ |  (Destination block)
   +------+
```
- **After Splitting**
```sql
   +------+
   |  BB  |  (Source block)
   +------+
      |
      v
   +-------+
   | NMBB  |  (New block inserted between BB and Succ)
   +-------+
      |
      v
   +------+
   | Succ |  (Destination block)
   +------+
```
#### Explanation

The diagram illustrates the **SplitCriticalEdge** process, a common transformation in compiler optimization and code generation. Splitting a critical edge introduces a new intermediate basic block to break a problematic edge in the control flow graph (CFG). This transformation ensures correctness for optimizations that rely on dominance or SSA (Static Single Assignment) form.

#### Key Points
1. **Critical Edge**:
   - A critical edge is an edge in the CFG where:
     - The **source block** (`BB`) has multiple successors.
     - The **destination block** (`Succ`) has multiple predecessors.

   Example:
   - Block `BB` has an edge to block `Succ`.
   - `BB` also branches to other blocks, and `Succ` has incoming edges from other blocks.

2. **Why Split the Edge?**
   - Critical edges complicate transformations because they can invalidate:
     - **Dominance relationships** (critical for correctness in SSA form).
     - **Liveness analysis** (used for register allocation and other optimizations).
   - By splitting the critical edge, you create a new block (`NMBB` - New Basic Block), simplifying transformations.

3. **How Splitting Works**:
   - A new block (`NMBB`) is inserted between `BB` (source) and `Succ` (destination).
   - Control flows from `BB` to `NMBB`, then to `Succ`.

4. **Post-Splitting Advantages**:
   - Dominance relationships are preserved.
   - Optimizations like instruction sinking or hoisting become easier to implement.
   - SSA form remains intact because `NMBB` isolates the edge.

---

### `performSink`
#### Diagram
- **Before Sinking**
```sql
   +--------------+
   | ParentBlock   |    (Contains MI)
   |      MI       |
   +--------------+
       /     \
      v       v
+------------------+
|  SuccToSinkTo    |   (Successor Block)
|  PHI instructions|   
|                  |   <- InsertPos
+------------------+
```

- **After Sinking**
```sql
   +--------------+
   | ParentBlock   |    (MI removed)
   +--------------+
       /     \            |
      |       |         splice
      v       v           v  
+------------------+      
|  SuccToSinkTo    |   (MI added after PHI instructions)
|  PHI instructions|
|       MI         |   <- InsertPos
+------------------+
```
```cpp
SuccToSinkTo.splice(InsertPos /*Where*/, ParentBlock, MI /*From*/, ++MachineBasicBlock::iterator(MI) /*TO*/);
```

#### Explanation

The diagram demonstrates the `performSink` function in LLVM, which moves a machine instruction (`MI`) from its original location in a **ParentBlock** to a **Successor Block** (`SuccToSinkTo`). This operation is called **sinking**, and it is performed to optimize code execution, typically by reducing register pressure and improving performance.

#### Key Points

1. **ParentBlock**:
   - This is the block where the instruction `MI` originally resides.

2. **SuccToSinkTo**:
   - A successor block where the instruction `MI` is moved to (sunk).
   - This block contains **PHI instructions**, which must remain at the top of the block for correctness in SSA form.

3. **Insert Position (`InsertPos`)**:
   - The sinking process ensures `MI` is inserted **after all PHI instructions** in the destination block (`SuccToSinkTo`).
   - The correct insertion point is determined by scanning for the first non-PHI instruction in `SuccToSinkTo`.

4. **Sinking Process**:
   - The instruction `MI` is **spliced** (moved) from the `ParentBlock` to the destination block (`SuccToSinkTo`).
   - The code snippet provided in the diagram shows how this is performed:
     ```cpp
     SuccToSinkTo.splice(InsertPos, ParentBlock, MI, ++MachineBasicBlock::iterator(MI));
     ```
     - **`InsertPos`**: Position in `SuccToSinkTo` where `MI` will be inserted.
     - **`ParentBlock`**: The block from which `MI` is being moved.
     - **`MI`**: The instruction being moved.
     - **`++iterator(MI)`**: Ensures that only `MI` is moved (not subsequent instructions).

---

### `SinkingPreventsImplicitNullCheck`
#### Diagram
- **Before Sinking** (Null Check Preserved)
```sql
+----------------+
|   PredMBB      |
| BNE $Reg, 0    |  (Branch: Checks if $Reg != 0)
+----------------+
       |
       v
+----------------+
|     MBB        |
| LD $Reg, Offset|  (Load: Safe if $Reg != 0)
+----------------+
```

- **After Sinking** (Null Check Broken)
```sql
+----------------+
|   PredMBB      |
| BNE $Reg, 0    |  (Branch: Checks if $Reg != 0)
+----------------+
       |
       v
+----------------+
|     MBB        |
+----------------+
       |
       v
+----------------+
| LD $Reg, Offset|  (Load: Potentially unsafe, $Reg could be 0)
+----------------+
```

```cpp
unsigned BaseReg;
int64_t Offset;
if (!TII->getMemOpBaseRegImmOfs(MI, BaseReg, Offset, TRI))
    return false;

if (!(MI.mayLoad() && MI.isPredicable()))
    return false;

MachineBranchPredicate MBP;
if (!TII->analyzeBranchPredicate(*PredMBB, MBP, false))
    return false;

return MBP.LHS.isReg() && MBP.RHS.isReg() && MBP.RHS.getImm() == 0 &&
       (MBP.Predicate == MachineBranchPredicate::PRED_NE ||
        MBP.Predicate == MachineBranchPredicate::PRED_EQ) &&
        MBP.LHS.getReg() == BaseReg;
```

#### Explanation

The image demonstrates a scenario where sinking an instruction (`MI`) from its current block (`MBB`) into a successor block may inadvertently **break an implicit null check** provided by a branch in the predecessor block (`PredMBB`). Here's how the updated code applies to the diagram:

#### 1. Components in the Diagram**

- **`PredMBB`**:
    - **Branch Instruction**: 
    - Contains a conditional branch (`BNE $Reg, 0`), which checks whether `$Reg` (the base register) is **not null** (`$Reg != 0`).
    - This acts as an **implicit null check**: If `$Reg` is `0`, the branch avoids executing the successor block that might otherwise cause a segmentation fault.

- **`MBB`**:
    - **Memory Load Instruction**:
    - Contains the instruction `LD $Reg, Offset`, which loads a value from memory using `$Reg` as the base address and an `Offset`.
    - This operation relies on `$Reg` being valid (non-null).

- **Implicit Null Check**:
    - The branch in `PredMBB` protects the load in `MBB`. If `$Reg == 0`, the branch prevents execution of `LD`, avoiding an invalid memory access.

---

#### 2. Sinking Problem

If the `LD $Reg, Offset` instruction (`MI`) is sunk from `MBB` to its successor, the null check provided by `BNE` in `PredMBB` is no longer effective, and `$Reg` may be dereferenced in paths where `$Reg == 0`. This causes undefined behavior.



#### 3. How the Updated Code Prevents Sinking

The updated code evaluates whether sinking `MI` would break the null check by performing several checks, as reflected in the flow of the image:

- **Step 1: Extract Base Register and Offset**
    ```cpp
    if (!TII->getMemOpBaseRegImmOfs(MI, BaseReg, Offset, TRI))
        return false;
    ```
    - The base register (`BaseReg`) and offset (`Offset`) of the memory instruction (`MI`) are extracted.
    - **From the Image**:
        - `$Reg` is identified as the base register for the memory load instruction `LD $Reg, Offset`.

- **Step 2: Ensure Load and Predicability**
    ```cpp
    if (!(MI.mayLoad() && MI.isPredicable()))
        return false;
    ```
    - Confirms that `MI`:
        1. Performs a **load operation** (`LD $Reg, Offset`).
        2. Can be conditionally executed (predicable).

- **Step 3: Analyze Branch in `PredMBB`**
    ```cpp
    if (!TII->analyzeBranchPredicate(*PredMBB, MBP, false))
        return false;
    ```
    - Analyzes the branch in `PredMBB` (`BNE $Reg, 0`) and extracts its predicate into `MBP`.

- **Step 4: Validate the Branch Predicate**
    ```cpp
    return MBP.LHS.isReg() && MBP.RHS.isImm() && MBP.RHS.getImm() == 0 &&
        (MBP.Predicate == MachineBranchPredicate::PRED_NE ||
            MBP.Predicate == MachineBranchPredicate::PRED_EQ) &&
            MBP.LHS.getReg() == BaseReg;
    ```
    - Checks that:
        1. **Left-Hand Side (`LHS`)**:
            - Is a register (`$Reg` in `BNE $Reg, 0`).
        2. **Right-Hand Side (`RHS`)**:
            - Is an immediate value `0`, indicating a null check.
        3. **Predicate**:
            - The branch checks whether `$Reg` is:
            - **Not equal to `0`** (`PRED_NE`): `$Reg != 0`.
            - **Equal to `0`** (`PRED_EQ`): `$Reg == 0`.
        4. **Register Match**:
            - The base register (`BaseReg`) of `LD $Reg, Offset` matches the register being checked in the branch (`$Reg`).



#### How the Code Prevents Sinking

1. **Guard Analysis**:
   - The branch in `PredMBB` acts as a null check for `$Reg`.
2. **Match Validation**:
   - Ensures that the base register in the load instruction is the same register being checked in the branch.
3. **Prevent Sinking**:
   - If the null check is valid and essential, sinking is disallowed, preserving the implicit protection provided by the branch.

---