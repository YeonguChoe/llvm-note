# CIRGenBuiltin.cpp

```cpp
  case Builtin::BI__builtin_fpclassify: {
    CIRGenFunction::CIRGenFPOptionsRAII fPOptsRAII(*this, e);
    mlir::Location loc = getLoc(e->getBeginLoc());
    mlir::Value value = emitScalarExpr(e->getArg(5));
    mlir::Type resultTy = convertType(e->getType());
    auto isZero =
        cir::IsFPClassOp::create(builder, loc, value, cir::FPClassTest::Zero);
    mlir::Value result =
        cir::TernaryOp::create(
            builder, loc, isZero,
            [&](mlir::OpBuilder &opBuilder, mlir::Location location) {
              mlir::Value zeroLiteral = emitScalarExpr(e->getArg(4));
              cir::YieldOp::create(opBuilder, location, zeroLiteral);
            },
            [&](mlir::OpBuilder &opBuilder, mlir::Location location) {
              auto isNan = cir::IsFPClassOp::create(opBuilder, location, value,
                                                    cir::FPClassTest::Nan);
              mlir::Value nanResult =
                  cir::TernaryOp::create(
                      opBuilder, location, isNan,
                      [&](mlir::OpBuilder &opBuilder, mlir::Location location) {
                        mlir::Value nanLiteral = emitScalarExpr(e->getArg(0));
                        cir::YieldOp::create(opBuilder, location, nanLiteral);
                      },
                      [&](mlir::OpBuilder &opBuilder, mlir::Location location) {
                        auto isInfinity = cir::IsFPClassOp::create(
                            opBuilder, location, value,
                            cir::FPClassTest::Infinity);
                        mlir::Value infResult =
                            cir::TernaryOp::create(
                                opBuilder, location, isInfinity,
                                [&](mlir::OpBuilder &opBuilder,
                                    mlir::Location location) {
                                  mlir::Value infinityLiteral =
                                      emitScalarExpr(e->getArg(1));
                                  cir::YieldOp::create(opBuilder, location,
                                                       infinityLiteral);
                                },
                                [&](mlir::OpBuilder &opBuilder,
                                    mlir::Location location) {
                                  auto isNormal = cir::IsFPClassOp::create(
                                      opBuilder, location, value,
                                      cir::FPClassTest::Normal);
                                  mlir::Value fpNormal =
                                      emitScalarExpr(e->getArg(2));
                                  mlir::Value fpSubnormal =
                                      emitScalarExpr(e->getArg(3));
                                  mlir::Value returnValue =
                                      cir::SelectOp::create(
                                          opBuilder, location, resultTy,
                                          isNormal, fpNormal, fpSubnormal);
                                  cir::YieldOp::create(opBuilder, location,
                                                       returnValue);
                                })
                                .getResult();
                        cir::YieldOp::create(opBuilder, location, infResult);
                      })
                      .getResult();
              cir::YieldOp::create(opBuilder, location, nanResult);
            })
            .getResult();
    return RValue::get(result);
  }
```

# builtin-fpclassify.c

```c
// RUN: %clang_cc1 -triple x86_64-unknown-linux-gnu -fclangir -emit-cir %s -o %t.cir
// RUN: FileCheck %s -check-prefix=CIR --input-file %t.cir
// RUN: %clang_cc1 -triple x86_64-unknown-linux-gnu -fclangir -emit-llvm %s -o %t-cir.ll
// RUN: FileCheck %s -check-prefix=LLVM --input-file %t-cir.ll
// RUN: %clang_cc1 -triple x86_64-unknown-linux-gnu -emit-llvm %s -o %t.ll
// RUN: FileCheck %s -check-prefix=OGCG --input-file %t.ll

#include <math.h>

void test_fpclassify_nan(){
    float nanValue = 0.0f / 0.0f;
    int nanResult = __builtin_fpclassify(FP_NAN, FP_INFINITE, FP_NORMAL,
                                         FP_SUBNORMAL, FP_ZERO, nanValue);
}

void test_fpclassify_inf(){
    float infValue = 1.0f / 0.0f;
    int infResult = __builtin_fpclassify(FP_NAN, FP_INFINITE, FP_NORMAL,
                                         FP_SUBNORMAL, FP_ZERO, infValue);
}

void test_fpclassify_normal(){
    float normalValue = 1.0f;
    int normalResult = __builtin_fpclassify(FP_NAN, FP_INFINITE, FP_NORMAL,
                                            FP_SUBNORMAL, FP_ZERO, normalValue);
}


void test_fpclassify_subnormal(){
    float subnormalValue = 1.0e-40f;
    int subnormalResult = __builtin_fpclassify(FP_NAN, FP_INFINITE, FP_NORMAL,
                                               FP_SUBNORMAL, FP_ZERO, subnormalValue);
}


void test_fpclassify_zero(){
    float zeroValue = 0.0f;
    int zeroResult = __builtin_fpclassify(FP_NAN, FP_INFINITE, FP_NORMAL,
                                          FP_SUBNORMAL, FP_ZERO, zeroValue);
}
```

