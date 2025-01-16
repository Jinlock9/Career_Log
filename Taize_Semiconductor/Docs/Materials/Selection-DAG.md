# Selection DAG

## The Data Structure
- SelectionDAGS are an alternative representation of the program
- Each SelectionDAG represents a signgle basic block

## SelectionDAG Types
### MVT
#### Definition
**Machine Value Type (MVT)** A union of the types that are supported by each target that uses SelectionDAG.
- Any given target will only support a subset of these types
- MVT examples include:
    - Integers: {i1, i32, i128, ...}
    - Floats: {f16, bf16, f80, ...}
    - Vectors: {v1i1, v2i32, v128bf16, ...}
    - Other things: {Other, Glue, ...}
### EVT
#### Definition
**Extended Value Type (EVT)** A union of the MVT types and all integer, float, and vectors types that LLVM IR supports.
- Does not include all LLVM IR types, struct and array types are not in the set
- EVT examples include:
    - All MVT types
    - Integer and vector lengths not natively supported on any architectures: {i3, v100i32, v99f32, v99i99, ...}

## SDNode
- These are the building blocks of all SelectionDAGs
- Each node has:
    - Opcode which defines the type
    - Potentially many other fields such as constant value or flags

### SDValue
- Represents the "output" of an SDNode
- Has an associated EVT
- SDValues have:
    - SDNode that defines this value
    - Index into the list of results from that node

### SDUse
- Represents the "input" to an SDNode
- SDUses have:
    - SDValue that is being used
    - SDNode that is the user
    - Operand index in the user node operand list

### SDNode - Revisited
- These are the building blocks of all SelectionDAGs
- Each node has:
    - Opcode which defines the type
    - 1 or more results represented by SDValues
    - O or more operands represented by SDUses
    - Potentially many other fields such as constant value or flags