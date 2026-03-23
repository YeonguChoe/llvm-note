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

#define FP_NAN       3
#define FP_INFINITE  516
#define FP_ZERO      96
#define FP_SUBNORMAL 144
#define FP_NORMAL    264

void test_fpclassify_nan(){
    float nanValue = 0.0f / 0.0f;
    __builtin_fpclassify(FP_NAN, FP_INFINITE, FP_NORMAL,
                                         FP_SUBNORMAL, FP_ZERO, nanValue);
// CIR: %[[IS_ZERO:.+]] = cir.is_fp_class %{{.+}}, fcZero : (!cir.float) -> !cir.bool
// CIR: cir.ternary(%[[IS_ZERO]], true {
// CIR: cir.const #cir.int<96> : !s32i
// CIR: %[[IS_NAN:.+]] = cir.is_fp_class %{{.+}}, fcNan : (!cir.float) -> !cir.bool
// CIR: cir.ternary(%[[IS_NAN]], true {
// CIR: cir.const #cir.int<3> : !s32i
// CIR: %[[IS_INF:.+]] = cir.is_fp_class %{{.+}}, fcInf : (!cir.float) -> !cir.bool
// CIR: cir.ternary(%[[IS_INF]], true {
// CIR: cir.const #cir.int<516> : !s32i
// CIR: %[[IS_NORMAL:.+]] = cir.is_fp_class %{{.+}}, fcNormal : (!cir.float) -> !cir.bool
// CIR: %[[NORMAL_VAL:.+]] = cir.const #cir.int<264> : !s32i
// CIR: %[[SUBNORMAL_VAL:.+]] = cir.const #cir.int<144> : !s32i
// CIR: cir.select if %[[IS_NORMAL]] then %[[NORMAL_VAL]] else %[[SUBNORMAL_VAL]] : (!cir.bool, !s32i, !s32i) -> !s32i

// LLVM: %[[VAL:.*]] = load float, ptr
// LLVM-NEXT: %[[IS_ZERO:.*]] = call i1 @llvm.is.fpclass.f32(float %[[VAL]], i32 96)
// LLVM-NEXT: br i1 %[[IS_ZERO]], label %[[BB_ZERO:.*]], label %[[BB_NOT_ZERO:.*]]
// LLVM: [[BB_ZERO]]:
// LLVM-NEXT: br label %[[BB_RET:.*]]
// LLVM: [[BB_NOT_ZERO]]:
// LLVM-NEXT: %[[IS_NAN:.*]] = call i1 @llvm.is.fpclass.f32(float %[[VAL]], i32 3)
// LLVM-NEXT: br i1 %[[IS_NAN]], label %[[BB_NAN:.*]], label %[[BB_NOT_NAN:.*]]
// LLVM: [[BB_NAN]]:
// LLVM-NEXT: br label %[[BB_MERGE1:.*]]
// LLVM: [[BB_NOT_NAN]]:
// LLVM-NEXT: %[[IS_INF:.*]] = call i1 @llvm.is.fpclass.f32(float %[[VAL]], i32 516)
// LLVM-NEXT: br i1 %[[IS_INF]], label %[[BB_INF:.*]], label %[[BB_NOT_INF:.*]]
// LLVM: [[BB_INF]]:
// LLVM-NEXT: br label %[[BB_MERGE2:.*]]
// LLVM: [[BB_NOT_INF]]:
// LLVM-NEXT: %[[IS_NORMAL:.*]] = call i1 @llvm.is.fpclass.f32(float %[[VAL]], i32 264)
// LLVM-NEXT: %[[NORMAL_OR_SUBNORMAL:.*]] = select i1 %[[IS_NORMAL]], i32 264, i32 144
// LLVM-NEXT: br label %[[BB_MERGE2]]
// LLVM: [[BB_MERGE2]]:
// LLVM-NEXT: %[[PHI_INF_SEL:.*]] = phi i32 [ %[[NORMAL_OR_SUBNORMAL]], %[[BB_NOT_INF]] ], [ 516, %[[BB_INF]] ]
// LLVM-NEXT: br label %[[BB_CONT1:.*]]
// LLVM: [[BB_CONT1]]:
// LLVM-NEXT: br label %[[BB_MERGE1]]
// LLVM: [[BB_MERGE1]]:
// LLVM-NEXT: %[[PHI_NAN_SEL:.*]] = phi i32 [ %[[PHI_INF_SEL]], %[[BB_CONT1]] ], [ 3, %[[BB_NAN]] ]
// LLVM-NEXT: br label %[[BB_CONT2:.*]]
// LLVM: [[BB_CONT2]]:
// LLVM-NEXT: br label %[[BB_RET]]
// LLVM: [[BB_RET]]:
// LLVM-NEXT: %[[PHI_FINAL:.*]] = phi i32 [ %[[PHI_NAN_SEL]], %[[BB_CONT2]] ], [ 96, %[[BB_ZERO]] ]
// LLVM-NEXT: br label %[[BB_EXIT:.*]]
// LLVM: [[BB_EXIT]]:
}

void test_fpclassify_inf(){
    float infValue = 1.0f / 0.0f;
    __builtin_fpclassify(FP_NAN, FP_INFINITE, FP_NORMAL,
                                         FP_SUBNORMAL, FP_ZERO, infValue);
// CIR: %[[IS_ZERO:.+]] = cir.is_fp_class %{{.+}}, fcZero : (!cir.float) -> !cir.bool
// CIR: cir.ternary(%[[IS_ZERO]], true {
// CIR: cir.const #cir.int<96> : !s32i
// CIR: %[[IS_NAN:.+]] = cir.is_fp_class %{{.+}}, fcNan : (!cir.float) -> !cir.bool
// CIR: cir.ternary(%[[IS_NAN]], true {
// CIR: cir.const #cir.int<3> : !s32i
// CIR: %[[IS_INF:.+]] = cir.is_fp_class %{{.+}}, fcInf : (!cir.float) -> !cir.bool
// CIR: cir.ternary(%[[IS_INF]], true {
// CIR: cir.const #cir.int<516> : !s32i
// CIR: %[[IS_NORMAL:.+]] = cir.is_fp_class %{{.+}}, fcNormal : (!cir.float) -> !cir.bool
// CIR: %[[NORMAL_VAL:.+]] = cir.const #cir.int<264> : !s32i
// CIR: %[[SUBNORMAL_VAL:.+]] = cir.const #cir.int<144> : !s32i
// CIR: cir.select if %[[IS_NORMAL]] then %[[NORMAL_VAL]] else %[[SUBNORMAL_VAL]] : (!cir.bool, !s32i, !s32i) -> !s32i

// LLVM: %[[VAL:.+]] = load float, ptr
// LLVM-NEXT: %[[IS_ZERO:.+]] = call i1 @llvm.is.fpclass.f32(float %[[VAL]], i32 96)
// LLVM-NEXT: br i1 %[[IS_ZERO]], label %[[BB_ZERO:.+]], label %[[BB_NOT_ZERO:.+]]
// LLVM: [[BB_ZERO]]:
// LLVM-NEXT: br label %[[BB_RET:.+]]
// LLVM: [[BB_NOT_ZERO]]:
// LLVM-NEXT: %[[IS_NAN:.+]] = call i1 @llvm.is.fpclass.f32(float %[[VAL]], i32 3)
// LLVM-NEXT: br i1 %[[IS_NAN]], label %[[BB_NAN:.+]], label %[[BB_NOT_NAN:.+]]
// LLVM: [[BB_NAN]]:
// LLVM-NEXT: br label %[[BB_MERGE1:.+]]
// LLVM: [[BB_NOT_NAN]]:
// LLVM-NEXT: %[[IS_INF:.+]] = call i1 @llvm.is.fpclass.f32(float %[[VAL]], i32 516)
// LLVM-NEXT: br i1 %[[IS_INF]], label %[[BB_INF:.+]], label %[[BB_NOT_INF:.+]]
// LLVM: [[BB_INF]]:
// LLVM-NEXT: br label %[[BB_MERGE2:.+]]
// LLVM: [[BB_NOT_INF]]:
// LLVM-NEXT: %[[IS_NORMAL:.+]] = call i1 @llvm.is.fpclass.f32(float %[[VAL]], i32 264)
// LLVM-NEXT: %[[SEL:.+]] = select i1 %[[IS_NORMAL]], i32 264, i32 144
// LLVM-NEXT: br label %[[BB_MERGE2]]
// LLVM: [[BB_MERGE2]]:
// LLVM-NEXT: %[[PHI1:.+]] = phi i32 [ %[[SEL]], %[[BB_NOT_INF]] ], [ 516, %[[BB_INF]] ]
// LLVM-NEXT: br label %[[BB_CONT1:.+]]
// LLVM: [[BB_CONT1]]:
// LLVM-NEXT: br label %[[BB_MERGE1]]
// LLVM: [[BB_MERGE1]]:
// LLVM-NEXT: %[[PHI2:.+]] = phi i32 [ %[[PHI1]], %[[BB_CONT1]] ], [ 3, %[[BB_NAN]] ]
// LLVM-NEXT: br label %[[BB_CONT2:.+]]
// LLVM: [[BB_CONT2]]:
// LLVM-NEXT: br label %[[BB_RET]]
// LLVM: [[BB_RET]]:
// LLVM-NEXT: %[[PHI3:.+]] = phi i32 [ %[[PHI2]], %[[BB_CONT2]] ], [ 96, %[[BB_ZERO]] ]
// LLVM-NEXT: br label
}

void test_fpclassify_normal(){
    float normalValue = 1.0f;
    __builtin_fpclassify(FP_NAN, FP_INFINITE, FP_NORMAL,
                                            FP_SUBNORMAL, FP_ZERO, normalValue);
// CIR: %[[IS_ZERO:.+]] = cir.is_fp_class %{{.+}}, fcZero : (!cir.float) -> !cir.bool
// CIR: cir.ternary(%[[IS_ZERO]], true {
// CIR: cir.const #cir.int<96> : !s32i
// CIR: %[[IS_NAN:.+]] = cir.is_fp_class %{{.+}}, fcNan : (!cir.float) -> !cir.bool
// CIR: cir.ternary(%[[IS_NAN]], true {
// CIR: cir.const #cir.int<3> : !s32i
// CIR: %[[IS_INF:.+]] = cir.is_fp_class %{{.+}}, fcInf : (!cir.float) -> !cir.bool
// CIR: cir.ternary(%[[IS_INF]], true {
// CIR: cir.const #cir.int<516> : !s32i
// CIR: %[[IS_NORMAL:.+]] = cir.is_fp_class %{{.+}}, fcNormal : (!cir.float) -> !cir.bool
// CIR: %[[NORMAL_VAL:.+]] = cir.const #cir.int<264> : !s32i
// CIR: %[[SUBNORMAL_VAL:.+]] = cir.const #cir.int<144> : !s32i
// CIR: cir.select if %[[IS_NORMAL]] then %[[NORMAL_VAL]] else %[[SUBNORMAL_VAL]] : (!cir.bool, !s32i, !s32i) -> !s32i

// LLVM: %[[VAL:.*]] = load float, ptr
// LLVM-NEXT: %[[IS_ZERO:.*]] = call i1 @llvm.is.fpclass.f32(float %[[VAL]], i32 96)
// LLVM-NEXT: br i1 %[[IS_ZERO]], label %[[BB_ZERO:.*]], label %[[BB_NOT_ZERO:.*]]
// LLVM: [[BB_ZERO]]:
// LLVM-NEXT: br label %[[BB_RET:.*]]
// LLVM: [[BB_NOT_ZERO]]:
// LLVM-NEXT: %[[IS_NAN:.*]] = call i1 @llvm.is.fpclass.f32(float %[[VAL]], i32 3)
// LLVM-NEXT: br i1 %[[IS_NAN]], label %[[BB_NAN:.*]], label %[[BB_NOT_NAN:.*]]
// LLVM: [[BB_NAN]]:
// LLVM-NEXT: br label %[[BB_MERGE1:.*]]
// LLVM: [[BB_NOT_NAN]]:
// LLVM-NEXT: %[[IS_INF:.*]] = call i1 @llvm.is.fpclass.f32(float %[[VAL]], i32 516)
// LLVM-NEXT: br i1 %[[IS_INF]], label %[[BB_INF:.*]], label %[[BB_NOT_INF:.*]]
// LLVM: [[BB_INF]]:
// LLVM-NEXT: br label %[[BB_MERGE2:.*]]
// LLVM: [[BB_NOT_INF]]:
// LLVM-NEXT: %[[IS_NORMAL:.*]] = call i1 @llvm.is.fpclass.f32(float %[[VAL]], i32 264)
// LLVM-NEXT: %[[NORMAL_OR_SUBNORMAL:.*]] = select i1 %[[IS_NORMAL]], i32 264, i32 144
// LLVM-NEXT: br label %[[BB_MERGE2]]
// LLVM: [[BB_MERGE2]]:
// LLVM-NEXT: %[[PHI_INF_SEL:.*]] = phi i32 [ %[[NORMAL_OR_SUBNORMAL]], %[[BB_NOT_INF]] ], [ 516, %[[BB_INF]] ]
// LLVM-NEXT: br label %[[BB_CONT1:.*]]
// LLVM: [[BB_CONT1]]:
// LLVM-NEXT: br label %[[BB_MERGE1]]
// LLVM: [[BB_MERGE1]]:
// LLVM-NEXT: %[[PHI_NAN_SEL:.*]] = phi i32 [ %[[PHI_INF_SEL]], %[[BB_CONT1]] ], [ 3, %[[BB_NAN]] ]
// LLVM-NEXT: br label %[[BB_CONT2:.*]]
// LLVM: [[BB_CONT2]]:
// LLVM-NEXT: br label %[[BB_RET]]
// LLVM: [[BB_RET]]:
// LLVM-NEXT: %[[PHI_FINAL:.*]] = phi i32 [ %[[PHI_NAN_SEL]], %[[BB_CONT2]] ], [ 96, %[[BB_ZERO]] ]
// LLVM-NEXT: br label %[[BB_EXIT:.*]]
// LLVM: [[BB_EXIT]]:
}

void test_fpclassify_subnormal(){
    float subnormalValue = 1.0e-40f;
    __builtin_fpclassify(FP_NAN, FP_INFINITE, FP_NORMAL,
                                               FP_SUBNORMAL, FP_ZERO, subnormalValue);
// CIR: %[[IS_ZERO:.+]] = cir.is_fp_class %{{.+}}, fcZero : (!cir.float) -> !cir.bool
// CIR: cir.ternary(%[[IS_ZERO]], true {
// CIR: cir.const #cir.int<96> : !s32i
// CIR: %[[IS_NAN:.+]] = cir.is_fp_class %{{.+}}, fcNan : (!cir.float) -> !cir.bool
// CIR: cir.ternary(%[[IS_NAN]], true {
// CIR: cir.const #cir.int<3> : !s32i
// CIR: %[[IS_INF:.+]] = cir.is_fp_class %{{.+}}, fcInf : (!cir.float) -> !cir.bool
// CIR: cir.ternary(%[[IS_INF]], true {
// CIR: cir.const #cir.int<516> : !s32i
// CIR: %[[IS_NORMAL:.+]] = cir.is_fp_class %{{.+}}, fcNormal : (!cir.float) -> !cir.bool
// CIR: %[[NORMAL_VAL:.+]] = cir.const #cir.int<264> : !s32i
// CIR: %[[SUBNORMAL_VAL:.+]] = cir.const #cir.int<144> : !s32i
// CIR: cir.select if %[[IS_NORMAL]] then %[[NORMAL_VAL]] else %[[SUBNORMAL_VAL]] : (!cir.bool, !s32i, !s32i) -> !s32i

// LLVM: %[[VAL:.*]] = load float, ptr
// LLVM-NEXT: %[[IS_ZERO:.*]] = call i1 @llvm.is.fpclass.f32(float %[[VAL]], i32 96)
// LLVM-NEXT: br i1 %[[IS_ZERO]], label %[[BB_ZERO:.*]], label %[[BB_NOT_ZERO:.*]]
// LLVM: [[BB_ZERO]]:
// LLVM-NEXT: br label %[[BB_RET:.*]]
// LLVM: [[BB_NOT_ZERO]]:
// LLVM-NEXT: %[[IS_NAN:.*]] = call i1 @llvm.is.fpclass.f32(float %[[VAL]], i32 3)
// LLVM-NEXT: br i1 %[[IS_NAN]], label %[[BB_NAN:.*]], label %[[BB_NOT_NAN:.*]]
// LLVM: [[BB_NAN]]:
// LLVM-NEXT: br label %[[BB_MERGE1:.*]]
// LLVM: [[BB_NOT_NAN]]:
// LLVM-NEXT: %[[IS_INF:.*]] = call i1 @llvm.is.fpclass.f32(float %[[VAL]], i32 516)
// LLVM-NEXT: br i1 %[[IS_INF]], label %[[BB_INF:.*]], label %[[BB_NOT_INF:.*]]
// LLVM: [[BB_INF]]:
// LLVM-NEXT: br label %[[BB_MERGE2:.*]]
// LLVM: [[BB_NOT_INF]]:
// LLVM-NEXT: %[[IS_NORMAL:.*]] = call i1 @llvm.is.fpclass.f32(float %[[VAL]], i32 264)
// LLVM-NEXT: %[[NORMAL_OR_SUBNORMAL:.*]] = select i1 %[[IS_NORMAL]], i32 264, i32 144
// LLVM-NEXT: br label %[[BB_MERGE2]]
// LLVM: [[BB_MERGE2]]:
// LLVM-NEXT: %[[PHI_INF_SEL:.*]] = phi i32 [ %[[NORMAL_OR_SUBNORMAL]], %[[BB_NOT_INF]] ], [ 516, %[[BB_INF]] ]
// LLVM-NEXT: br label %[[BB_CONT1:.*]]
// LLVM: [[BB_CONT1]]:
// LLVM-NEXT: br label %[[BB_MERGE1]]
// LLVM: [[BB_MERGE1]]:
// LLVM-NEXT: %[[PHI_NAN_SEL:.*]] = phi i32 [ %[[PHI_INF_SEL]], %[[BB_CONT1]] ], [ 3, %[[BB_NAN]] ]
// LLVM-NEXT: br label %[[BB_CONT2:.*]]
// LLVM: [[BB_CONT2]]:
// LLVM-NEXT: br label %[[BB_RET]]
// LLVM: [[BB_RET]]:
// LLVM-NEXT: %[[PHI_FINAL:.*]] = phi i32 [ %[[PHI_NAN_SEL]], %[[BB_CONT2]] ], [ 96, %[[BB_ZERO]] ]
// LLVM-NEXT: br label %[[BB_EXIT:.*]]
// LLVM: [[BB_EXIT]]:
}

void test_fpclassify_zero(){
    float zeroValue = 0.0f;
    __builtin_fpclassify(FP_NAN, FP_INFINITE, FP_NORMAL,
                                          FP_SUBNORMAL, FP_ZERO, zeroValue);
// CIR: %[[IS_ZERO:.+]] = cir.is_fp_class %{{.+}}, fcZero : (!cir.float) -> !cir.bool
// CIR: cir.ternary(%[[IS_ZERO]], true {
// CIR: cir.const #cir.int<96> : !s32i
// CIR: %[[IS_NAN:.+]] = cir.is_fp_class %{{.+}}, fcNan : (!cir.float) -> !cir.bool
// CIR: cir.ternary(%[[IS_NAN]], true {
// CIR: cir.const #cir.int<3> : !s32i
// CIR: %[[IS_INF:.+]] = cir.is_fp_class %{{.+}}, fcInf : (!cir.float) -> !cir.bool
// CIR: cir.ternary(%[[IS_INF]], true {
// CIR: cir.const #cir.int<516> : !s32i
// CIR: %[[IS_NORMAL:.+]] = cir.is_fp_class %{{.+}}, fcNormal : (!cir.float) -> !cir.bool
// CIR: %[[NORMAL_VAL:.+]] = cir.const #cir.int<264> : !s32i
// CIR: %[[SUBNORMAL_VAL:.+]] = cir.const #cir.int<144> : !s32i
// CIR: cir.select if %[[IS_NORMAL]] then %[[NORMAL_VAL]] else %[[SUBNORMAL_VAL]] : (!cir.bool, !s32i, !s32i) -> !s32i

// LLVM: %[[VAL:.*]] = load float, ptr
// LLVM-NEXT: %[[IS_ZERO:.*]] = call i1 @llvm.is.fpclass.f32(float %[[VAL]], i32 96)
// LLVM-NEXT: br i1 %[[IS_ZERO]], label %[[BB_ZERO:.*]], label %[[BB_NOT_ZERO:.*]]
// LLVM: [[BB_ZERO]]:
// LLVM-NEXT: br label %[[BB_RET:.*]]
// LLVM: [[BB_NOT_ZERO]]:
// LLVM-NEXT: %[[IS_NAN:.*]] = call i1 @llvm.is.fpclass.f32(float %[[VAL]], i32 3)
// LLVM-NEXT: br i1 %[[IS_NAN]], label %[[BB_NAN:.*]], label %[[BB_NOT_NAN:.*]]
// LLVM: [[BB_NAN]]:
// LLVM-NEXT: br label %[[BB_MERGE1:.*]]
// LLVM: [[BB_NOT_NAN]]:
// LLVM-NEXT: %[[IS_INF:.*]] = call i1 @llvm.is.fpclass.f32(float %[[VAL]], i32 516)
// LLVM-NEXT: br i1 %[[IS_INF]], label %[[BB_INF:.*]], label %[[BB_NOT_INF:.*]]
// LLVM: [[BB_INF]]:
// LLVM-NEXT: br label %[[BB_MERGE2:.*]]
// LLVM: [[BB_NOT_INF]]:
// LLVM-NEXT: %[[IS_NORMAL:.*]] = call i1 @llvm.is.fpclass.f32(float %[[VAL]], i32 264)
// LLVM-NEXT: %[[NORMAL_OR_SUBNORMAL:.*]] = select i1 %[[IS_NORMAL]], i32 264, i32 144
// LLVM-NEXT: br label %[[BB_MERGE2]]
// LLVM: [[BB_MERGE2]]:
// LLVM-NEXT: %[[PHI_INF_SEL:.*]] = phi i32 [ %[[NORMAL_OR_SUBNORMAL]], %[[BB_NOT_INF]] ], [ 516, %[[BB_INF]] ]
// LLVM-NEXT: br label %[[BB_CONT1:.*]]
// LLVM: [[BB_CONT1]]:
// LLVM-NEXT: br label %[[BB_MERGE1]]
// LLVM: [[BB_MERGE1]]:
// LLVM-NEXT: %[[PHI_NAN_SEL:.*]] = phi i32 [ %[[PHI_INF_SEL]], %[[BB_CONT1]] ], [ 3, %[[BB_NAN]] ]
// LLVM-NEXT: br label %[[BB_CONT2:.*]]
// LLVM: [[BB_CONT2]]:
// LLVM-NEXT: br label %[[BB_RET]]
// LLVM: [[BB_RET]]:
// LLVM-NEXT: %[[PHI_FINAL:.*]] = phi i32 [ %[[PHI_NAN_SEL]], %[[BB_CONT2]] ], [ 96, %[[BB_ZERO]] ]
// LLVM-NEXT: br label %[[BB_EXIT:.*]]
// LLVM: [[BB_EXIT]]:
}
```

