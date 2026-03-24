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

```
