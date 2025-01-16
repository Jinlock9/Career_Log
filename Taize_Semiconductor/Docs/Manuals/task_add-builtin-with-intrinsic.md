# Task
- Date: 2025.01.15 ~ 2025.01.16

### Description
To add a new builtin called `set_trace`, with corresponding intrinsics into company's backend.

Refer to `Set Trace` from company's individual instructions.

### Procedure

Commit: [`f015d6f`](https://github.com/ChipLTech/LLVM/commit/f015d6f51a6cdf332e19b9424fd6f06655abd68a)

### Compilation
```sh
# Compile ----------------
# go LLVM/build dir
ninja
```

### Test
- `/LLVM/build/test`

- Create test file:

    ```C
    // test_set_trace.c

    int main() {
        int test_value = 42;
        set_trace(test_value);

        return 0;
    }

- Compile to .ll file

    ```sh
    ../bin/clang -target dlc -std=dc99 -emit-llvm -O2 -c test_set_trace.c -o test_set_trace.ll
    # or
    ../bin/clang -target dlc -std=dc99 -emit-llvm -O2 -S test_set_trace.c -o test_set_trace.ll
    ```

### Reference
- Builtin Rule: `LLVM/clang/include/clang/Basic/Builtins.def`
- ISD Opcodes: `LLVM/llvm/include/llvm/CodeGen/ISDOpcodes.h`

### Commands
```sh
find [path] -name "file_name"
grep -rl 'content' [path]
```
