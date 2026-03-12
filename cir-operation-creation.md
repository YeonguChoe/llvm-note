# Creating ClangIR operation (CIR -> LLVM)
- Website: https://llvm.github.io/clangir/Dialect/ops.html

- Operations: [cir.acos, cir.asin, cir.atan2, cir.atan, cir.abs, cir.alloc.exception, cir.alloca, cir.array.ctor, cir.array.dtor, cir.assume.aligned, cir.assume, cir.assume.separate_storage, cir.atomic.clear, cir.atomic.cmp_xchg, cir.atomic.fence, cir.atomic.fetch, cir.atomic.test_and_set, cir.atomic.xchg, cir.await, cir.base_class_addr, cir.base_data_member, cir.base_method, cir.binop, cir.binop.overflow, cir.clrsb, cir.clz, cir.ctz, cir.ffs, cir.parity, cir.popcount, cir.bit_reverse, cir.blockaddress, cir.brcond, cir.br, cir.break, cir.byte_swap, cir.call, cir.case, cir.cast, cir.catch_param, cir.ceil, cir.clear_cache, cir.cmp, cir.cmp3way, cir.complex.binop, cir.complex.create, cir.complex.imag, cir.complex.imag_ptr, cir.complex.real, cir.complex.real_ptr, cir.condition, cir.const, cir.continue, cir.copy, cir.copysign, cir.cos, cir.delete.array, cir.derived_class_addr, cir.derived_data_member, cir.derived_method, cir.do, cir.dyn_cast, cir.eh.inflight_exception, cir.eh.setjmp, cir.eh.typeid, cir.exp2, cir.exp, cir.expect, cir.extract_member, cir.fabs, cir.fmaxnum, cir.fmaximum, cir.fminnum, cir.fminimum, cir.fmod, cir.floor, cir.for, cir.frame_address, cir.free.exception, cir.get_bitfield, cir.get_element, cir.get_global, cir.get_member, cir.get_method, cir.get_runtime_member, cir.global, cir.goto, cir.if, cir.asm, cir.insert_member, cir.invariant_group, cir.is_constant, cir.is_fp_class, cir.std.begin, cir.std.end, cir.llvm.intrinsic, cir.llrint, cir.llround, cir.label, cir.linker_options, cir.load, cir.log10, cir.log2, cir.log, cir.lrint, cir.lround, cir.libc.memchr, cir.memcpy_inline, cir.libc.memcpy, cir.libc.memmove, cir.memset_inline, cir.libc.memset, cir.nearbyint, cir.objsize, cir.pow, cir.prefetch, cir.ptr_diff, cir.ptr_mask, cir.ptr_stride, cir.resume, cir.return, cir.rint, cir.rotate, cir.roundeven, cir.round, cir.scope, cir.select, cir.set_bitfield, cir.shift, cir.signbit, cir.sin, cir.sqrt, cir.stack_restore, cir.stack_save, cir.std.find, cir.store, cir.switch.flat, cir.switch, cir.tan, cir.ternary, cir.throw, cir.trap, cir.trunc, cir.try_call, cir.try, cir.unary, cir.unreachable, cir.va.arg, cir.va.copy, cir.va.end, cir.va.start, cir.vtt.address_point, cir.vtable.address_point, cir.vtable.get_vptr, cir.vtable.get_virtual_fn_addr, cir.vec.cmp, cir.vec.create, cir.vec.extract, cir.vec.insert, cir.vec.shuffle.dynamic, cir.vec.shuffle, cir.vec.splat, cir.vec.ternary, cir.while, cir.yield, cir.func]

## `CIROps.td`

- location: `/clang/include/clang/CIR/Dialect/IR/CIROps.td`

```cpp

```

## `CIRBaseBuilder.h`

- location: `clang/include/clang/CIR/Dialect/Builder/CIRBaseBuilder.h`

```cpp

```

## `LowerToLLVM.cpp`

- location: `clang/lib/CIR/Lowering/DirectToLLVM/LowerToLLVMIR.cpp`

```cpp

```
