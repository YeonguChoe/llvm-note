# signbit function


```cpp
CIRGenFunction::CIRGenFPOptionsRAII FPOptsRAII(*this,e);
mlir::Location loc = getLoc(e->getBeginLoc());
mlir::Value v= emitScalarExpr(e->getArg(0));
mlir::Value signbit = emitSignBit(loc, *this, v);
return RValue::get(builder.createBoolToInt(signbit, convertType(e->getType())));
```


