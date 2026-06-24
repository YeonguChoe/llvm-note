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
- Only use 64 bit register: `d<d>, d<n>`

```python
if x.bit_length < 64:
    ANY_EXT to 64 bits
elif x.bit_length > 64:
    .clampScalar(0, s64, s64)
    .clampScalar(1, s64, s64)
```

### 3. `llvm/utils/update_mir_test_checks.py` Testing
- llvm/test/CodeGen/AArch64/GlobalISel/combine-zext-shl.ll
- llvm/test/CodeGen/AArch64/GlobalISel/split-wide-shifts-multiway.ll
- llvm/test/CodeGen/AArch64/shift.ll
- llvm/test/CodeGen/AArch64/select_const.ll
- llvm/test/CodeGen/AArch64/hoist-and-by-const-from-shl-in-eqcmp-zero.ll
- llvm/test/CodeGen/AArch64/funnel-shift.ll
- llvm/test/CodeGen/AArch64/fsh.ll
- llvm/test/CodeGen/ARM/frem-power2.ll
