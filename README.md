# CIR practice

## Command
- Converting C/C++ to CIR

```bash
/tmp/install-llvm/bin/clang -fclangir -emit-cir -clangir-disable-passes  main.c -o main.cir
```

- Converting CIR to LLVM IR

```bash
/tmp/install-llvm/bin/cir-translate --cir-to-llvmir main.cir -o main.ll
```
