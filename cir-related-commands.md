# ClangIR Related commands

#### `C/C++` -> `ClangIR`

```bash
clang -fclangir -emit-cir <C/C++ file>
```

#### `ClangIR` -> `MLIR Generic form`

```bash
cir-opt <CIR file> --mlir-print-op-generic -o <MLIR file>
```

#### `ClangIR` -> `LLVM IR`

```bash
cir-translate --cir-to-llvmir <CIR file> -o <LLVM file>
```
