
# GitHub
- https://github.com/llvm/llvm-project

# Build

## LLVM
```bash
cmake -S llvm \
    -B build \
    -G Ninja \
    -DCMAKE_BUILD_TYPE=Debug \
    -DCMAKE_C_COMPILER=clang \
    -DCMAKE_CXX_COMPILER=clang++ \
    -DLLVM_USE_LINKER=lld \
    -DLLVM_ENABLE_PROJECTS="clang;lld" \
    -DLLVM_TARGETS_TO_BUILD="AArch64;ARM;RISCV;X86" \
    -DLLVM_ENABLE_ASSERTIONS=ON
```

```bash
ninja -j$(( $(nproc) / 2 ))
```

## Search File

```bash
find . -name "*<file>*"
```

# Reset to Original Clone

## Reset Repository
```bash
git reset --hard HEAD
```

## Reset Repository Including Untracked Files

```bash
git clean -df
```

## Reset Specific File
```bash
git restore <file>
```

# Test

```bash
build/bin/llvm-lit <test file>
```

# C/C++ -> LLVM IR

```bash
clang -S -emit-llvm <C/C++ file> -o <LLVM IR file>
```

### Format LLVM-Project Code

```bash
clang-format -i -style=llvm <file>
```
