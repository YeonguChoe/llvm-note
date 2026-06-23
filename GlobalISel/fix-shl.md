# Fix shl

- Stats: https://davemgreen.github.io/gisel.html

```
LLVM IR ---> Legalizer ---> Instruction Select ---> MIR
```

## 1. LLVM IR range
- bit count: [1, 2^23]
- Reference: `llvm/include/llvm/IR/DerivedTypes.h:52`

## 2. AArch Register size limit

### Scalar
- Use only 64 bit register: `d<d>, d<n>`

### Vector
- 

