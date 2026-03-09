# Clang's fpclassify

- Order: FP_NAN, FP_INFINITE, FP_NORMAL, FP_SUBNORMAL, FP_ZERO

### Files
- clang/lib/CodeGen/CGBuiltin.cpp

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
- clang/test/CodeGen/strictfp_builtins.c
  - Line 159: `#[[ATTR5:[0-9]+]]`
  - test_fpclass
 
```c
// CHECK-LABEL: @test_fpclassify(
// CHECK-NEXT:  entry:
// CHECK-NEXT:    [[D_ADDR:%.*]] = alloca double, align 8
// CHECK-NEXT:    store double [[D:%.*]], ptr [[D_ADDR]], align 8
// CHECK-NEXT:    [[TMP0:%.*]] = load double, ptr [[D_ADDR]], align 8
// CHECK-NEXT:    [[TMP1:%.*]] = call i1 @llvm.is.fpclass.f64(double [[TMP0]], i32 3) #[[ATTR:[0-9]+]]
// CHECK-NEXT:    br i1 [[TMP1]], label [[FPCLASSIFY_END:%.*]], label [[FPCLASSIFY_NOT_NAN:%.*]]
// CHECK:       fpclassify_not_nan:
// CHECK-NEXT:    [[TMP2:%.*]] = call i1 @llvm.is.fpclass.f64(double [[TMP0]], i32 516) #[[ATTR]]
// CHECK-NEXT:    br i1 [[TMP2]], label [[FPCLASSIFY_END]], label [[FPCLASSIFY_NOT_INF:%.*]]
// CHECK:       fpclassify_not_inf:
// CHECK-NEXT:    [[TMP3:%.*]] = call i1 @llvm.is.fpclass.f64(double [[TMP0]], i32 264) #[[ATTR]]
// CHECK-NEXT:    br i1 [[TMP3]], label [[FPCLASSIFY_END]], label [[FPCLASSIFY_NOT_NORMAL:%.*]]
// CHECK:       fpclassify_not_normal:
// CHECK-NEXT:    [[TMP4:%.*]] = call i1 @llvm.is.fpclass.f64(double [[TMP0]], i32 144) #[[ATTR]]
// CHECK-NEXT:    br i1 [[TMP4]], label [[FPCLASSIFY_END]], label [[FPCLASSIFY_NOT_SUBNORMAL:%.*]]
// CHECK:       fpclassify_not_subnormal:
// CHECK-NEXT:    br label [[FPCLASSIFY_END]]
// CHECK:       fpclassify_end:
// CHECK-NEXT:    [[FPCLASSIFY_RESULT:%.*]] = phi i32 [ 0, [[ENTRY:%.*]] ], [ 1, [[FPCLASSIFY_NOT_NAN]] ], [ 2, [[FPCLASSIFY_NOT_INF]] ], [ 3, [[FPCLASSIFY_NOT_NORMAL]] ], [ 4, [[FPCLASSIFY_NOT_SUBNORMAL]] ]
// CHECK-NEXT:    call void @p(ptr noundef @.str.1, i32 noundef [[FPCLASSIFY_RESULT]]) #[[ATTR]]
// CHECK-NEXT:    ret void
//
void test_fpclassify(double d) {
  P(fpclassify, (0, 1, 2, 3, 4, d));
  return;
}
```

- builtin.c

```c
  resld = __builtin_fabsl(LD);
  // CHECK: call float @llvm.fabs.f32(float
  // CHECK: call double @llvm.fabs.f64(double
  // CHECK: call [[LDTYPE]] @llvm.fabs.[[LDLLVMTY:[a-z0-9]+]]([[LDTYPE]]
```

```c
  ul = __builtin_bswapg(ul);
  // CHECK: call [[LDLLVMTY:[a-z0-9]+]] @llvm.bswap.[[LONGINTTY]]
```



- clang/test/Headers/nvptx_device_math_macro.cpp
