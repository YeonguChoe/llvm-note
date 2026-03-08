# Clang's fpclassify

- Order: FP_NAN, FP_INFINITE, FP_NORMAL, FP_SUBNORMAL, FP_ZERO

### Files
- clang/lib/CodeGen/CGBuiltin.cpp
- clang/test/CodeGen/strictfp_builtins.c
- clang/test/Headers/nvptx_device_math_macro.cpp

```cpp
  case Builtin::BI__builtin_fpclassify: {
    CodeGenFunction::CGFPOptionsRAII FPOptsRAII(*this, E);

    Value *NanLiteral = EmitScalarExpr(E->getArg(0));
    Value *InfiniteLiteral = EmitScalarExpr(E->getArg(1));
    Value *NormalLiteral = EmitScalarExpr(E->getArg(2));
    Value *SubnormalLiteral = EmitScalarExpr(E->getArg(3));
    Value *ZeroLiteral = EmitScalarExpr(E->getArg(4));
    Value *V = EmitScalarExpr(E->getArg(5));

    BasicBlock *Entry = Builder.GetInsertBlock();
    BasicBlock *NotNan = createBasicBlock("fpclassify_not_nan", CurFn);
    BasicBlock *NotInfinite = createBasicBlock("fpclassify_not_inf", CurFn);
    BasicBlock *NotNormal = createBasicBlock("fpclassify_not_normal", CurFn);
    BasicBlock *NotSubnormal =
        createBasicBlock("fpclassify_not_subnormal", CurFn);
    BasicBlock *End = createBasicBlock("fpclassify_end", CurFn);

    // End block
    Builder.SetInsertPoint(End);
    PHINode *Result = Builder.CreatePHI(ConvertType(E->getArg(0)->getType()), 5,
                                        "fpclassify_result");

    // Entry block
    Builder.SetInsertPoint(Entry);
    Value *IsNan = Builder.createIsFPClass(V, FPClassTest::fcNan);
    Result->addIncoming(NanLiteral, Entry);
    Builder.CreateCondBr(IsNan, End, NotNan);

    // NotNan block
    Builder.SetInsertPoint(NotNan);
    Value *IsInf = Builder.createIsFPClass(V, FPClassTest::fcInf);
    Result->addIncoming(InfiniteLiteral, NotNan);
    Builder.CreateCondBr(IsInf, End, NotInfinite);

    // NotInfinite block
    Builder.SetInsertPoint(NotInfinite);
    Value *IsNormal = Builder.createIsFPClass(V, FPClassTest::fcNormal);
    Result->addIncoming(NormalLiteral, NotInfinite);
    Builder.CreateCondBr(IsNormal, End, NotNormal);

    // NotNormal block
    Builder.SetInsertPoint(NotNormal);
    Value *IsSubnormal = Builder.createIsFPClass(V, FPClassTest::fcSubnormal);
    Result->addIncoming(SubnormalLiteral, NotNormal);
    Builder.CreateCondBr(IsSubnormal, End, NotSubnormal);

    // NotSubnormal block
    Builder.SetInsertPoint(NotSubnormal);
    Result->addIncoming(ZeroLiteral, NotSubnormal);
    Builder.CreateBr(End);

    // End block
    Builder.SetInsertPoint(End);
    return RValue::get(Result);
  }
```

```python
# Order: FP_NAN, FP_INFINITE, FP_NORMAL, FP_SUBNORMAL, FP_ZERO
def fp_classify():
    is_nan = True
    if is_nan:
        nan_literal = 1
        return nan_literal
    else:
        is_infinity = True
        if is_infinity:
            infinity_literal = 2
            return infinity_literal
        else:
            is_normal = True
            if is_normal:
                normal_literal = 3
                return normal_literal
            else:
                is_subnormal = True
                if is_subnormal:
                    subnormal_literal = 4
                    return subnormal_literal
                else:
                    zero_literal = 5
                    return zero_literal

```
