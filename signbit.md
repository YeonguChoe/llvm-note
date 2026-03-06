# signbit function


```cpp
CIRGenFunction::CIRGenFPOptionsRAII FPOptsRAII(*this,e);
mlir::Location loc = getLoc(e->getBeginLoc());
mlir::Value v= emitScalarExpr(e->getArg(0));
mlir::Value signbit = emitSignBit(loc, *this, v);
return RValue::get(builder.createBoolToInt(signbit, convertType(e->getType())));
```


### test

```c
// RUN: %clang_cc1 -fclangir -emit-cir %s -o %t.cir
// RUN: FileCheck --input-file=%t.cir %s -check-prefix=CIR
// RUN: %clang_cc1 -fclangir -emit-llvm %s -o %t.cir.ll
// RUN: FileCheck --input-file=%t.cir.ll %s -check-prefix=LLVM
// RUN: %clang_cc1 -emit-llvm %s -o %t.ll
// RUN: FileCheck --input-file=%t.ll %s -check-prefix=OGCG

int test_signbit_float(float x) {
  return __builtin_signbit(x);
}
// CIR: cir.signbit %{{.*}} : !cir.float -> !cir.bool
// LLVM: bitcast float {{.*}} to i32
// LLVM: icmp slt i32 {{.*}}, 0
// LLVM: zext i1 {{.*}} to i32
// LLVM: ret i32
// OGCG: call i1 @llvm.is.fpclass.f32(float {{.*}}, i32 128)


int test_signbit_double(double x) {
  return __builtin_signbit(x);
}
// CIR: cir.signbit %{{.*}} : !cir.double -> !cir.bool
// LLVM:       bitcast double {{.*}} to i64
// LLVM:       icmp slt i64 {{.*}}, 0
// LLVM:       zext i1 {{.*}} to i32
// LLVM:       ret i32
// OGCG:       call i1 @llvm.is.fpclass.f64(double {{.*}}, i32 128)

int test_signbit_long_double(long double x) {
  return __builtin_signbit(x);
}
// CIR: cir.signbit %{{.*}} : !cir.long_double<!cir.f80> -> !cir.bool
// LLVM: call i1 @llvm.is.fpclass.f80(x86_fp80 {{.*}}, i32 128)
// OGCG: call i1 @llvm.is.fpclass.f80(x86_fp80 {{.*}}, i32 128)

int test_signbitf(float x) {
  return __builtin_signbitf(x);
}
// CIR: cir.signbit %{{.*}} : !cir.float -> !cir.bool
// LLVM: call i1 @llvm.is.fpclass.f32(float {{.*}}, i32 128)
// OGCG: call i1 @llvm.is.fpclass.f32(float {{.*}}, i32 128)

int test_signbitl(long double x) {
  return __builtin_signbitl(x);
}
// CIR: cir.signbit %{{.*}} : !cir.long_double<!cir.f80> -> !cir.bool
// LLVM: call i1 @llvm.is.fpclass.f80(x86_fp80 {{.*}}, i32 128)
// OGCG: call i1 @llvm.is.fpclass.f80(x86_fp80 {{.*}}, i32 128)
```
