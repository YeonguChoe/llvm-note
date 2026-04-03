
# GitHub
- https://github.com/llvm/llvm-project

# Build

```bash
cmake -S llvm \
    -B build \
    -G Ninja \
    -DCMAKE_BUILD_TYPE=Debug \
    -DLLVM_ENABLE_ASSERTIONS=ON \
    -DCMAKE_C_COMPILER=clang \
    -DCMAKE_CXX_COMPILER=clang++ \
    -DLLVM_USE_LINKER=lld \
    -DLLVM_TARGETS_TO_BUILD="host"
```

```bash
ninja -C build clang
```

```bash
ninja -j9 && ninja check-clang -j9
```

# Test

```bash
build/bin/llvm-lit ../clang/test/CIR/<test file>
```

# Test: C/C++ -> CIR -> LLVM IR
```bash
clang -fclangir -emit-cir <C/C++ file> -o - | cir-translate --cir-to-llvmir - -o <LLVM IR file>
```

# Lower C/C++ into CIR file

```bash
clang -fclangir -emit-cir <C/C++ file>
```

# C/C++ -> LLVM IR

```bash
clang -S -emit-llvm <C/C++ file> -o <LLVM IR file>
```

### Format LLVM-Project Code

```bash
clang-format -i -style=llvm <file>
```

## File to change

- clang/lib/CIR/Lowering/DirectToLLVM/LowerToLLVM.cpp
- clang/include/clang/CIR/Dialect/Builder/CIRBaseBuilder.h
- clang/include/clang/CIR/Dialect/IR/CIROps.td

# CIR practice

### CIR Project

```cpp
  case Builtin::BI__builtin_fpclassify: {
    CIRGenFunction::CIRGenFPOptionsRAII FPOptsRAII(*this, e);
    mlir::Location Loc = getLoc(e->getBeginLoc());

    mlir::Value NanLiteral = emitScalarExpr(e->getArg(0));
    mlir::Value InfinityLiteral = emitScalarExpr(e->getArg(1));
    mlir::Value NormalLiteral = emitScalarExpr(e->getArg(2));
    mlir::Value SubnormalLiteral = emitScalarExpr(e->getArg(3));
    mlir::Value ZeroLiteral = emitScalarExpr(e->getArg(4));
    mlir::Value V = emitScalarExpr(e->getArg(5));

    mlir::Type ResultTy = convertType(e->getType());
    mlir::Block *EntryBlock = builder.getInsertionBlock();
    mlir::Region *Region = EntryBlock->getParent();

    // Create Blocks
    mlir::Block *InfinityBlock = builder.createBlock(Region, Region->end());
    mlir::Block *NormalBlock = builder.createBlock(Region, Region->end());
    mlir::Block *SubnormalBlock = builder.createBlock(Region, Region->end());
    mlir::Block *ZeroBlock = builder.createBlock(Region, Region->end());
    mlir::Block *EndBlock = builder.createBlock(Region, Region->end());
    EndBlock->addArgument(ResultTy, Loc);

    // ^EntryBlock
    builder.setInsertionPointToEnd(EntryBlock);
    mlir::Value IsNan = builder.createIsFPClass(Loc, V, cir::FPClassTest::Nan);
    cir::BrCondOp::create(builder, Loc, IsNan, EndBlock,
                          InfinityBlock, // destTrue, destFalse
                          mlir::ValueRange{NanLiteral},
                          mlir::ValueRange{}); // operandsTrue, operandsFalse

    // ^InfinityBlock
    builder.setInsertionPointToEnd(InfinityBlock);
    mlir::Value IsInfinity =
        builder.createIsFPClass(Loc, V, cir::FPClassTest::Infinity);
    cir::BrCondOp::create(builder, Loc, IsInfinity, EndBlock, NormalBlock,
                          mlir::ValueRange{InfinityLiteral},
                          mlir::ValueRange{});

    // ^NormalBlock
    builder.setInsertionPointToEnd(NormalBlock);
    mlir::Value IsNormal =
        builder.createIsFPClass(Loc, V, cir::FPClassTest::Normal);
    cir::BrCondOp::create(builder, Loc, IsNormal, EndBlock, SubnormalBlock,
                          mlir::ValueRange{NormalLiteral}, mlir::ValueRange{});

    // ^SubnormalBlock
    builder.setInsertionPointToEnd(SubnormalBlock);
    mlir::Value IsSubnormal =
        builder.createIsFPClass(Loc, V, cir::FPClassTest::Subnormal);
    cir::BrCondOp::create(builder, Loc, IsSubnormal, EndBlock, ZeroBlock,
                          mlir::ValueRange{SubnormalLiteral},
                          mlir::ValueRange{});

    // ^ZeroBlock
    builder.setInsertionPointToEnd(ZeroBlock);
    cir::BrOp::create(builder, Loc, EndBlock, mlir::ValueRange{ZeroLiteral});

    // ^EndBlock(x)
    builder.setInsertionPointToEnd(EndBlock);
    mlir::Value Result = EndBlock->getArgument(0);

    return RValue::get(Result);
  }
```

### LLVM project

```cpp
  case Builtin::BI__builtin_fpclassify: {
    CodeGenFunction::CGFPOptionsRAII FPOptsRAII(*this, E);
    Value *V = EmitScalarExpr(E->getArg(5));

    Value *IsNan = Builder.createIsFPClass(V, FPClassTest::fcNan);
    Value *IsInf = Builder.createIsFPClass(V, FPClassTest::fcInf);
    Value *IsNormal = Builder.createIsFPClass(V, FPClassTest::fcNormal);
    Value *IsSubnormal = Builder.createIsFPClass(V, FPClassTest::fcSubnormal);

    BasicBlock *Entry = Builder.GetInsertBlock();

    BasicBlock *End = createBasicBlock("fpclassify_end", CurFn);
    Builder.SetInsertPoint(End);
    PHINode *Result = Builder.CreatePHI(ConvertType(E->getArg(0)->getType()), 5,
                                        "fpclassify_result");

    // Check if V is NaN
    Builder.SetInsertPoint(Entry);
    BasicBlock *NotNan = createBasicBlock("fpclassify_not_nan", CurFn);
    Builder.CreateCondBr(IsNan, End, NotNan);
    Value *NanLiteral = EmitScalarExpr(E->getArg(0));
    Result->addIncoming(NanLiteral, Entry);

    // Check if V is infinity
    Builder.SetInsertPoint(NotNan);
    BasicBlock *NotInf = createBasicBlock("fpclassify_not_inf", CurFn);
    Builder.CreateCondBr(IsInf, End, NotInf);
    Value *InfLiteral = EmitScalarExpr(E->getArg(1));
    Result->addIncoming(InfLiteral, NotNan);

    // Check if V is normal
    Builder.SetInsertPoint(NotInf);
    BasicBlock *NotNormal = createBasicBlock("fpclassify_not_normal", CurFn);
    Builder.CreateCondBr(IsNormal, End, NotNormal);
    Value *NormalLiteral = EmitScalarExpr(E->getArg(2));
    Result->addIncoming(NormalLiteral, NotInf);

    // Check if V is subnormal
    Builder.SetInsertPoint(NotNormal);
    BasicBlock *NotSubnormal =
        createBasicBlock("fpclassify_not_subnormal", CurFn);
    Builder.CreateCondBr(IsSubnormal, End, NotSubnormal);
    Value *SubnormalLiteral = EmitScalarExpr(E->getArg(3));
    Result->addIncoming(SubnormalLiteral, NotNormal);

    // If V is not one of the above, it is zero
    Builder.SetInsertPoint(NotSubnormal);
    Builder.CreateBr(End);
    Value *ZeroLiteral = EmitScalarExpr(E->getArg(4));
    Result->addIncoming(ZeroLiteral, NotSubnormal);

    Builder.SetInsertPoint(End);
    return RValue::get(Result);
  }
```

- Test case

```c
// CHECK-LABEL: @test_fpclassify(
// CHECK-NEXT:  entry:
// CHECK-NEXT:    [[D_ADDR:%.*]] = alloca double, align 8
// CHECK-NEXT:    store double [[D:%.*]], ptr [[D_ADDR]], align 8
// CHECK-NEXT:    [[TMP0:%.*]] = load double, ptr [[D_ADDR]], align 8
// CHECK-NEXT:    [[TMP1:%.*]] = call i1 @llvm.is.fpclass.f64(double [[TMP0]], i32 3) #[[ATTR4]]
// CHECK-NEXT:    [[TMP2:%.*]] = call i1 @llvm.is.fpclass.f64(double [[TMP0]], i32 516) #[[ATTR4]]
// CHECK-NEXT:    [[TMP3:%.*]] = call i1 @llvm.is.fpclass.f64(double [[TMP0]], i32 264) #[[ATTR4]]
// CHECK-NEXT:    [[TMP4:%.*]] = call i1 @llvm.is.fpclass.f64(double [[TMP0]], i32 144) #[[ATTR4]]
// CHECK-NEXT:    br i1 [[TMP1]], label [[FPCLASSIFY_END:%.*]], label [[FPCLASSIFY_NOT_NAN:%.*]]
// CHECK:       fpclassify_end:
// CHECK-NEXT:    [[FPCLASSIFY_RESULT:%.*]] = phi i32 [ 0, [[ENTRY:%.*]] ], [ 1, [[FPCLASSIFY_NOT_NAN]] ], [ 2, [[FPCLASSIFY_NOT_INF:%.*]] ], [ 3, [[FPCLASSIFY_NOT_NORMAL:%.*]] ], [ 4, [[FPCLASSIFY_NOT_SUBNORMAL:%.*]] ]
// CHECK-NEXT:    call void @p(ptr noundef @.str.1, i32 noundef [[FPCLASSIFY_RESULT]]) #[[ATTR4]]
// CHECK-NEXT:    ret void
// CHECK:       fpclassify_not_nan:
// CHECK-NEXT:    br i1 [[TMP2]], label [[FPCLASSIFY_END]], label [[FPCLASSIFY_NOT_INF]]
// CHECK:       fpclassify_not_inf:
// CHECK-NEXT:    br i1 [[TMP3]], label [[FPCLASSIFY_END]], label [[FPCLASSIFY_NOT_NORMAL]]
// CHECK:       fpclassify_not_normal:
// CHECK-NEXT:    br i1 [[TMP4]], label [[FPCLASSIFY_END]], label [[FPCLASSIFY_NOT_SUBNORMAL]]
// CHECK:       fpclassify_not_subnormal:
// CHECK-NEXT:    br label [[FPCLASSIFY_END]]
//
void test_fpclassify(double d) {
  P(fpclassify, (0, 1, 2, 3, 4, d));
  return;
}

// CHECK-LABEL: @test_isinf_sign(
// CHECK-NEXT:  entry:
// CHECK-NEXT:    [[D_ADDR:%.*]] = alloca double, align 8
// CHECK-NEXT:    store double [[D:%.*]], ptr [[D_ADDR]], align 8
// CHECK-NEXT:    [[TMP0:%.*]] = load double, ptr [[D_ADDR]], align 8
// CHECK-NEXT:    [[TMP1:%.*]] = call double @llvm.fabs.f64(double [[TMP0]]) #[[ATTR5:[0-9]+]]
// CHECK-NEXT:    [[ISINF:%.*]] = call i1 @llvm.experimental.constrained.fcmp.f64(double [[TMP1]], double 0x7FF0000000000000, metadata !"oeq", metadata !"fpexcept.strict") #[[ATTR4]]
// CHECK-NEXT:    [[TMP2:%.*]] = bitcast double [[TMP0]] to i64
// CHECK-NEXT:    [[TMP3:%.*]] = icmp slt i64 [[TMP2]], 0
// CHECK-NEXT:    [[TMP4:%.*]] = select i1 [[TMP3]], i32 -1, i32 1
// CHECK-NEXT:    [[TMP5:%.*]] = select i1 [[ISINF]], i32 [[TMP4]], i32 0
// CHECK-NEXT:    call void @p(ptr noundef @.str.8, i32 noundef [[TMP5]]) #[[ATTR4]]
// CHECK-NEXT:    ret void
//
void test_isinf_sign(double d) {
  P(isinf_sign, (d));

  return;
}
```

### Command
#### Build C/C++ file to executable file
```bash
clang -fclangir main.c -o main
```

#### Build CIR file
```bash
clang -fclangir -emit-cir main.c -o main.cir
```

#### Build LLVM IR file
```bash
clang -fclangir -emit-llvm -S main.c -o main.ll
```

#### Build LLVM IR file to executable file
```bash
clang main.ll -o main
```
