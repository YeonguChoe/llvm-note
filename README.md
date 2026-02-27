# Build

```bash
cmake -S llvm -B build -G Ninja -DCMAKE_BUILD_TYPE=Release -DLLVM_ENABLE_PROJECTS="clang;mlir" -DLLVM_TARGETS_TO_BUILD="host" -DLLVM_ENABLE_ASSERTIONS=ON -DCLANG_ENABLE_CIR=ON -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++ -DLLVM_USE_LINKER=lld
```

```bash
ninja -C build clang
```

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

- Converting C/C++ to LLVM IR

```bash
/tmp/install-llvm/bin/clang -fclangir -S -emit-llvm main.c
```

```cpp
  case Builtin::BI__builtin_fpclassify: {
    CIRGenFunction::CIRGenFPOptionsRAII FPOptsRAII(*this, E);
    mlir::Location Loc = getLoc(E->getBeginLoc());

    mlir::Value NanLiteral = emitScalarExpr(E->getArg(0));
    mlir::Value InfLiteral = emitScalarExpr(E->getArg(1));
    mlir::Value NormalLiteral = emitScalarExpr(E->getArg(2));
    mlir::Value SubnormalLiteral = emitScalarExpr(E->getArg(3));
    mlir::Value ZeroLiteral = emitScalarExpr(E->getArg(4));
    mlir::Value V = emitScalarExpr(E->getArg(5));

    mlir::Block *EntryBlock = builder.getInsertionBlock();
    mlir::Region *Region = EntryBlock->getParent();

    mlir::Block *InfBlock = builder.createBlock(Region, Region->end());
    mlir::Block *NormalBlock = builder.createBlock(Region, Region->end());
    mlir::Block *SubnormalBlock = builder.createBlock(Region, Region->end());
    mlir::Block *ZeroBlock = builder.createBlock(Region, Region->end());
    
    mlir::Block *EndBlock = builder.createBlock(Region, Region->end());
    mlir::Type ResultTy = ConvertType(E->getType());
    EndBlock->addArgument(ResultTy, Loc);

    // ^Entry: if NaN -> End(NanLiteral), else -> InfBlock
    builder.setInsertionPointToEnd(EntryBlock);
    mlir::Value IsNan = builder.createIsFPClass(Loc, V, FPClassTest::fcNan);
    builder.create<mlir::cir::BrCondOp>(Loc, IsNan, EndBlock,
                                           mlir::ValueRange{NanLiteral},
                                           InfBlock, mlir::ValueRange{});

    // ^InfBlock: if Inf -> End(InfLiteral), else -> NormalBlock
    builder.setInsertionPointToEnd(InfBlock);
    mlir::Value IsInf = builder.createIsFPClass(Loc, V, FPClassTest::fcInf);
    builder.create<mlir::cir::BrCondOp>(Loc, IsInf, EndBlock,
                                           mlir::ValueRange{InfLiteral},
                                           NormalBlock, mlir::ValueRange{});

    // ^NormalBlock: if Normal -> End(NormalLiteral), else -> SubnormalBlock
    builder.setInsertionPointToEnd(NormalBlock);
    mlir::Value IsNormal =
        builder.createIsFPClass(Loc, V, FPClassTest::fcNormal);
    builder.create<mlir::cir::BrCondOp>(Loc, IsNormal, EndBlock,
                                           mlir::ValueRange{NormalLiteral},
                                           SubnormalBlock, mlir::ValueRange{});

    // ^SubnormalBlock: if Subnormal -> End(SubnormalLiteral), else -> ZeroBlock
    builder.setInsertionPointToEnd(SubnormalBlock);
    mlir::Value IsSubnormal =
        builder.createIsFPClass(Loc, V, FPClassTest::fcSubnormal);
    builder.create<mlir::cir::BrCondOp>(Loc, IsSubnormal, EndBlock,
                                           mlir::ValueRange{SubnormalLiteral},
                                           ZeroBlock, mlir::ValueRange{});

    // ^ZeroBlock: unconditionally -> End(ZeroLiteral)
    builder.setInsertionPointToEnd(ZeroBlock);
    builder.create<mlir::cir::BrOp>(Loc, EndBlock,
                                       mlir::ValueRange{ZeroLiteral});

    // ^EndBlock(x)
    builder.setInsertionPointToEnd(EndBlock);
    mlir::Value Result = EndBlock->getArgument(0);

    return RValue::get(Result);
  }
```

```cpp
    CodeGenFunction::CGFPOptionsRAII FPOptsRAII(*this, E);
    Value *V = EmitScalarExpr(E->getArg(5));

    Value *NanLiteral = EmitScalarExpr(E->getArg(0));
    Value *InfLiteral = EmitScalarExpr(E->getArg(1));
    Value *NormalLiteral = EmitScalarExpr(E->getArg(2));
    Value *SubnormalLiteral = EmitScalarExpr(E->getArg(3));
    Value *ZeroLiteral = EmitScalarExpr(E->getArg(4));

    Value *IsNan = Builder.createIsFPClass(V, 0b0000000011);
    Value *IsInf = Builder.createIsFPClass(V, 0b1000000100);
    Value *IsNormal = Builder.createIsFPClass(V, 0b0100001000);
    Value *IsSubnormal = Builder.createIsFPClass(V, 0b0010010000);

    BasicBlock *Entry = Builder.GetInsertBlock();

    BasicBlock *End = createBasicBlock("fpclassify_end", CurFn);
    Builder.SetInsertPoint(End);
    PHINode *Result = Builder.CreatePHI(ConvertType(E->getArg(0)->getType()), 5,
                                        "fpclassify_result");

    // Check if V is NaN
    Builder.SetInsertPoint(Entry);
    BasicBlock *NotNan = createBasicBlock("fpclassify_not_nan", CurFn);
    Builder.CreateCondBr(IsNan, End, NotNan);
    Result->addIncoming(NanLiteral, Entry);

    // Check if V is infinity
    Builder.SetInsertPoint(NotNan);
    BasicBlock *NotInf = createBasicBlock("fpclassify_not_inf", CurFn);
    Builder.CreateCondBr(IsInf, End, NotInf);
    Result->addIncoming(InfLiteral, NotNan);

    // Check if V is normal
    Builder.SetInsertPoint(NotInf);
    BasicBlock *NotNormal = createBasicBlock("fpclassify_not_normal", CurFn);
    Builder.CreateCondBr(IsNormal, End, NotNormal);
    Result->addIncoming(NormalLiteral, NotInf);

    // Check if V is subnormal
    Builder.SetInsertPoint(NotNormal);
    BasicBlock *NotSubnormal =
        createBasicBlock("fpclassify_not_subnormal", CurFn);
    Builder.CreateCondBr(IsSubnormal, End, NotSubnormal);
    Result->addIncoming(SubnormalLiteral, NotNormal);

    // If V is not one of the above, it is zero
    Builder.SetInsertPoint(NotSubnormal);
    Builder.CreateBr(End);
    Result->addIncoming(ZeroLiteral, NotSubnormal);

    Builder.SetInsertPoint(End);
    return RValue::get(Result);
```
