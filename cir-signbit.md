# Builtin SignBit


```cpp
  case Builtin::BI__builtin_signbitl: {
    CIRGenFunction::CIRGenFPOptionsRAII fPOptsRAII(*this, e);
    mlir::Location loc = getLoc(e->getBeginLoc());
    mlir::Value value = emitScalarExpr(e->getArg(0));
    mlir::Type resultTy = convertType(e->getType());
    mlir::Value signBit = emitSignBit(loc, *this, value);
    return RValue::get(builder.createBoolToInt(signBit, resultTy));
  }
```

## builtin-signbit.c

```c
// RUN: %clang_cc1 -triple x86_64-unknown-linux-gnu -fclangir -emit-cir %s -o %t.cir
// RUN: FileCheck %s --check-prefix=CIR --input-file %t.cir

void test_signbit_positive_zero(){
  double positiveZero = +0.0;
  __builtin_signbit(positiveZero);
// CIR: %[[ALLOCA:.*]] = cir.alloca !cir.double
// CIR: %[[CONST:.*]] = cir.const #cir.fp<{{.*}}> : !cir.double
// CIR: cir.store align({{[0-9]+}}) %[[CONST]], %[[ALLOCA]] : !cir.double, !cir.ptr<!cir.double>
}

void test_signbit_negative_zero(){
  double negativeZero = -0.0;
  __builtin_signbit(negativeZero);
// CIR: %[[ALLOCA:.*]] = cir.alloca !cir.double
// CIR: %[[CONST:.*]] = cir.const #cir.fp<{{.*}}> : !cir.double
// CIR: cir.store align({{[0-9]+}}) %[[CONST]], %[[ALLOCA]] : !cir.double, !cir.ptr<!cir.double>
}

void test_signbit_positive_number(){
  double positiveNumber = 1.0;
  __builtin_signbit(positiveNumber);
// CIR: %[[ALLOCA:.*]] = cir.alloca !cir.double
// CIR: %[[CONST:.*]] = cir.const #cir.fp<{{.*}}> : !cir.double
// CIR: cir.store align({{[0-9]+}}) %[[CONST]], %[[ALLOCA]] : !cir.double, !cir.ptr<!cir.double>
}

void test_signbit_negative_number(){
  double negativeNumber = -1.0;
  __builtin_signbit(negativeNumber);
// CIR: %[[ALLOCA:.*]] = cir.alloca !cir.double
// CIR: %[[CONST:.*]] = cir.const #cir.fp<{{.*}}> : !cir.double
// CIR: cir.store align({{[0-9]+}}) %[[CONST]], %[[ALLOCA]] : !cir.double, !cir.ptr<!cir.double>
}

void test_signbit_positive_nan(){
  double positiveNan = +__builtin_nan("");
  __builtin_signbit(positiveNan);
// CIR: %[[ALLOCA:.*]] = cir.alloca !cir.double
// CIR: %[[CONST:.*]] = cir.const #cir.fp<{{.*}}> : !cir.double
// CIR: cir.store align({{[0-9]+}}) %[[CONST]], %[[ALLOCA]] : !cir.double, !cir.ptr<!cir.double>
}

void test_signbit_negative_nan(){
  double negativeNan = -__builtin_nan("");
  __builtin_signbit(negativeNan);
// CIR: %[[ALLOCA:.*]] = cir.alloca !cir.double
// CIR: %[[CONST:.*]] = cir.const #cir.fp<{{.*}}> : !cir.double
// CIR: cir.store align({{[0-9]+}}) %[[CONST]], %[[ALLOCA]] : !cir.double, !cir.ptr<!cir.double>
}

void test_signbit_positive_infinity(){
  double positiveInfinity = +__builtin_inf();
  __builtin_signbit(positiveInfinity);
// CIR: %[[ALLOCA:.*]] = cir.alloca !cir.double
// CIR: %[[CONST:.*]] = cir.const #cir.fp<{{.*}}> : !cir.double
// CIR: cir.store align({{[0-9]+}}) %[[CONST]], %[[ALLOCA]] : !cir.double, !cir.ptr<!cir.double>
}

void test_signbit_negative_infinity(){
  double negativeInfinity = -__builtin_inf();
  __builtin_signbit(negativeInfinity);
// CIR: %[[ALLOCA:.*]] = cir.alloca !cir.double
// CIR: %[[CONST:.*]] = cir.const #cir.fp<{{.*}}> : !cir.double
// CIR: cir.store align({{[0-9]+}}) %[[CONST]], %[[ALLOCA]] : !cir.double, !cir.ptr<!cir.double>
}
```
