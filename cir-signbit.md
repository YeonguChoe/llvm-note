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
// RUN: %clang_cc1 -triple x86_64-unknown-linux-gnu -fclangir -emit-llvm %s -o %t-cir.ll
// RUN: FileCheck %s --check-prefix=LLVM --input-file %t-cir.ll
// RUN: %clang_cc1 -triple x86_64-unknown-linux-gnu -emit-llvm %s -o %t.ll
// RUN: FileCheck %s --check-prefix=OGCG --input-file %t.ll

void test_signbit_positive_zero(){
  double positiveZero = +0.0;
  int result = __builtin_signbit(positiveZero);
// CIR: %[[ALLOCA:.*]] = cir.alloca !cir.double
// CIR: %[[CONST:.*]] = cir.const #cir.fp<{{.*}}> : !cir.double
// CIR: cir.store align({{[0-9]+}}) %[[CONST]], %[[ALLOCA]] : !cir.double, !cir.ptr<!cir.double>

// LLVM: %[[V4:[0-9]+]] = load double, ptr %[[V2:[0-9]+]]
// LLVM: %[[V5:[0-9]+]] = bitcast double %[[V4]] to i[[#BITS:]]
// LLVM: %[[V6:[0-9]+]] = icmp slt i[[#BITS]] %[[V5]], 0
// LLVM: %[[V7:[0-9]+]] = zext i1 %[[V6]] to i{{[0-9]+}}

// OGCG: %{{[0-9]+}} = load double, ptr %[[V0:[a-zA-Z0-9]+]]
// OGCG: %{{[0-9]+}} = bitcast double %{{[0-9]+}} to i{{[0-9]+}}
// OGCG: %{{[0-9]+}} = icmp slt i{{[0-9]+}} %{{[0-9]+}}, 0
// OGCG: %{{[0-9]+}} = zext i1 %{{[0-9]+}} to i{{[0-9]+}}
}

void test_signbit_negative_zero(){
  double negativeZero = -0.0;
  int result = __builtin_signbit(negativeZero);
// CIR: %[[ALLOCA:.*]] = cir.alloca !cir.double
// CIR: %[[CONST:.*]] = cir.const #cir.fp<{{.*}}> : !cir.double
// CIR: cir.store align({{[0-9]+}}) %[[CONST]], %[[ALLOCA]] : !cir.double, !cir.ptr<!cir.double>

// LLVM: %[[V4:[0-9]+]] = load double, ptr %[[V2:[0-9]+]]
// LLVM: %[[V5:[0-9]+]] = bitcast double %[[V4]] to i[[#BITS:]]
// LLVM: %[[V6:[0-9]+]] = icmp slt i[[#BITS]] %[[V5]], 0
// LLVM: %[[V7:[0-9]+]] = zext i1 %[[V6]] to i{{[0-9]+}}

// OGCG: %{{[0-9]+}} = load double, ptr %[[V0:[a-zA-Z0-9]+]]
// OGCG: %{{[0-9]+}} = bitcast double %{{[0-9]+}} to i{{[0-9]+}}
// OGCG: %{{[0-9]+}} = icmp slt i{{[0-9]+}} %{{[0-9]+}}, 0
// OGCG: %{{[0-9]+}} = zext i1 %{{[0-9]+}} to i{{[0-9]+}}
}

void test_signbit_positive_number(){
  double positiveNumber = 1.0;
  int result = __builtin_signbit(positiveNumber);
// CIR: %[[ALLOCA:.*]] = cir.alloca !cir.double
// CIR: %[[CONST:.*]] = cir.const #cir.fp<{{.*}}> : !cir.double
// CIR: cir.store align({{[0-9]+}}) %[[CONST]], %[[ALLOCA]] : !cir.double, !cir.ptr<!cir.double>

// LLVM: %[[V4:[0-9]+]] = load double, ptr %[[V2:[0-9]+]]
// LLVM: %[[V5:[0-9]+]] = bitcast double %[[V4]] to i[[#BITS:]]
// LLVM: %[[V6:[0-9]+]] = icmp slt i[[#BITS]] %[[V5]], 0
// LLVM: %[[V7:[0-9]+]] = zext i1 %[[V6]] to i{{[0-9]+}}

// OGCG: %{{[0-9]+}} = load double, ptr %[[V0:[a-zA-Z0-9]+]]
// OGCG: %{{[0-9]+}} = bitcast double %{{[0-9]+}} to i{{[0-9]+}}
// OGCG: %{{[0-9]+}} = icmp slt i{{[0-9]+}} %{{[0-9]+}}, 0
// OGCG: %{{[0-9]+}} = zext i1 %{{[0-9]+}} to i{{[0-9]+}}
}

void test_signbit_negative_number(){
  double negativeNumber = -1.0;
  int result = __builtin_signbit(negativeNumber);
// CIR: %[[ALLOCA:.*]] = cir.alloca !cir.double
// CIR: %[[CONST:.*]] = cir.const #cir.fp<{{.*}}> : !cir.double
// CIR: cir.store align({{[0-9]+}}) %[[CONST]], %[[ALLOCA]] : !cir.double, !cir.ptr<!cir.double>

// LLVM: %[[V4:[0-9]+]] = load double, ptr %[[V2:[0-9]+]]
// LLVM: %[[V5:[0-9]+]] = bitcast double %[[V4]] to i[[#BITS:]]
// LLVM: %[[V6:[0-9]+]] = icmp slt i[[#BITS]] %[[V5]], 0
// LLVM: %[[V7:[0-9]+]] = zext i1 %[[V6]] to i{{[0-9]+}}

// OGCG: %{{[0-9]+}} = load double, ptr %[[V0:[a-zA-Z0-9]+]]
// OGCG: %{{[0-9]+}} = bitcast double %{{[0-9]+}} to i{{[0-9]+}}
// OGCG: %{{[0-9]+}} = icmp slt i{{[0-9]+}} %{{[0-9]+}}, 0
// OGCG: %{{[0-9]+}} = zext i1 %{{[0-9]+}} to i{{[0-9]+}}
}

void test_signbit_positive_nan(){
  double positiveNan = +__builtin_nan("");
  int result = __builtin_signbit(positiveNan);
// CIR: %[[ALLOCA:.*]] = cir.alloca !cir.double
// CIR: %[[CONST:.*]] = cir.const #cir.fp<{{.*}}> : !cir.double
// CIR: cir.store align({{[0-9]+}}) %[[CONST]], %[[ALLOCA]] : !cir.double, !cir.ptr<!cir.double>

// LLVM: %[[V4:[0-9]+]] = load double, ptr %[[V2:[0-9]+]]
// LLVM: %[[V5:[0-9]+]] = bitcast double %[[V4]] to i[[#BITS:]]
// LLVM: %[[V6:[0-9]+]] = icmp slt i[[#BITS]] %[[V5]], 0
// LLVM: %[[V7:[0-9]+]] = zext i1 %[[V6]] to i{{[0-9]+}}

// OGCG: %{{[0-9]+}} = load double, ptr %[[V0:[a-zA-Z0-9]+]]
// OGCG: %{{[0-9]+}} = bitcast double %{{[0-9]+}} to i{{[0-9]+}}
// OGCG: %{{[0-9]+}} = icmp slt i{{[0-9]+}} %{{[0-9]+}}, 0
// OGCG: %{{[0-9]+}} = zext i1 %{{[0-9]+}} to i{{[0-9]+}}
}

void test_signbit_negative_nan(){
  double negativeNan = -__builtin_nan("");
  int result = __builtin_signbit(negativeNan);
// CIR: %[[ALLOCA:.*]] = cir.alloca !cir.double
// CIR: %[[CONST:.*]] = cir.const #cir.fp<{{.*}}> : !cir.double
// CIR: cir.store align({{[0-9]+}}) %[[CONST]], %[[ALLOCA]] : !cir.double, !cir.ptr<!cir.double>

// LLVM: %[[V4:[0-9]+]] = load double, ptr %[[V2:[0-9]+]]
// LLVM: %[[V5:[0-9]+]] = bitcast double %[[V4]] to i[[#BITS:]]
// LLVM: %[[V6:[0-9]+]] = icmp slt i[[#BITS]] %[[V5]], 0
// LLVM: %[[V7:[0-9]+]] = zext i1 %[[V6]] to i{{[0-9]+}}

// OGCG: %{{[0-9]+}} = load double, ptr %[[V0:[a-zA-Z0-9]+]]
// OGCG: %{{[0-9]+}} = bitcast double %{{[0-9]+}} to i{{[0-9]+}}
// OGCG: %{{[0-9]+}} = icmp slt i{{[0-9]+}} %{{[0-9]+}}, 0
// OGCG: %{{[0-9]+}} = zext i1 %{{[0-9]+}} to i{{[0-9]+}}
}

void test_signbit_positive_infinity(){
  double positiveInfinity = +__builtin_inf();
  int result = __builtin_signbit(positiveInfinity);
// CIR: %[[ALLOCA:.*]] = cir.alloca !cir.double
// CIR: %[[CONST:.*]] = cir.const #cir.fp<{{.*}}> : !cir.double
// CIR: cir.store align({{[0-9]+}}) %[[CONST]], %[[ALLOCA]] : !cir.double, !cir.ptr<!cir.double>

// LLVM: %[[V4:[0-9]+]] = load double, ptr %[[V2:[0-9]+]]
// LLVM: %[[V5:[0-9]+]] = bitcast double %[[V4]] to i[[#BITS:]]
// LLVM: %[[V6:[0-9]+]] = icmp slt i[[#BITS]] %[[V5]], 0
// LLVM: %[[V7:[0-9]+]] = zext i1 %[[V6]] to i{{[0-9]+}}

// OGCG: %{{[0-9]+}} = load double, ptr %[[V0:[a-zA-Z0-9]+]]
// OGCG: %{{[0-9]+}} = bitcast double %{{[0-9]+}} to i{{[0-9]+}}
// OGCG: %{{[0-9]+}} = icmp slt i{{[0-9]+}} %{{[0-9]+}}, 0
// OGCG: %{{[0-9]+}} = zext i1 %{{[0-9]+}} to i{{[0-9]+}}
}

void test_signbit_negative_infinity(){
  double negativeInfinity = -__builtin_inf();
  int result = __builtin_signbit(negativeInfinity);
// CIR: %[[ALLOCA:.*]] = cir.alloca !cir.double
// CIR: %[[CONST:.*]] = cir.const #cir.fp<{{.*}}> : !cir.double
// CIR: cir.store align({{[0-9]+}}) %[[CONST]], %[[ALLOCA]] : !cir.double, !cir.ptr<!cir.double>

// LLVM: %[[V4:[0-9]+]] = load double, ptr %[[V2:[0-9]+]]
// LLVM: %[[V5:[0-9]+]] = bitcast double %[[V4]] to i[[#BITS:]]
// LLVM: %[[V6:[0-9]+]] = icmp slt i[[#BITS]] %[[V5]], 0
// LLVM: %[[V7:[0-9]+]] = zext i1 %[[V6]] to i{{[0-9]+}}

// OGCG: %{{[0-9]+}} = load double, ptr %[[V0:[a-zA-Z0-9]+]]
// OGCG: %{{[0-9]+}} = bitcast double %{{[0-9]+}} to i{{[0-9]+}}
// OGCG: %{{[0-9]+}} = icmp slt i{{[0-9]+}} %{{[0-9]+}}, 0
// OGCG: %{{[0-9]+}} = zext i1 %{{[0-9]+}} to i{{[0-9]+}}
}
```
