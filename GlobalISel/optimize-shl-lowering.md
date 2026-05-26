# Optimize shl lowering by removing G_ZEXT

## llvm/lib/Target/AArch64/AArch64Combine.td

- generate code at 

```cpp
// before:
//   %b32:_(s32) = G_ZEXT %b:_(s8)
//   %r:_(s32)   = G_SHL  %a:_(s32), %b32:_(s32)
// after:
//   %r:_(s32)   = G_SHL  %a:_(s32), (G_ANYEXT %b)
def remove_zext_when_lowering_shl : GICombineRule<
  (defs root:$root, register_matchinfo:$matchinfo),
  (match (wip_match_opcode G_SHL):$root,
         [{ return matchRemoveZextWhenLoweringShift(*${root}, MRI, ${matchinfo}); }]),
  (apply [{ applyRemoveZextWhenLoweringShift(*${root}, MRI, B, ${matchinfo}); }])
>;

// Post-legalization combines which are primarily optimizations.
def AArch64PostLegalizerCombiner
    : GICombiner<"AArch64PostLegalizerCombinerImpl",
                       [copy_prop, cast_of_cast_combines, constant_fold_fp_ops,
                        ...
                        remove_zext_when_lowering_shl,
```

## llvm/lib/Target/AArch64/GISel/AArch64PostLegalizerCombiner.cpp

```cpp
bool matchRemoveZextWhenLoweringShl(MachineInstr &MI, MachineRegisterInfo &MRI,
                                    Register &MatchInfo) {
  // check Opcode is G_SHL
  assert(MI.getOpcode() == TargetOpcode::G_SHL && "Opcode is not G_SHL");

  // check the shape of operation is correct

  // check destination register is 32 bit or 64 bit scalar type
  LLT DstTy = MRI.getType(MI.getOperand(0).getReg());
  if (DstTy.isVector()) {
    return false;
  }
  unsigned BW = DstTy.getScalarSizeInBits();
  if (BW != 32 && BW != 64) {
    return false;
  }

  // check amount register originated from G_ZEXT
  Register AmtReg = MI.getOperand(2).getReg();
  MachineInstr *AmtDef = MRI.getVRegDef(AmtReg);
  if (!AmtDef || AmtDef->getOpcode() != TargetOpcode::G_ZEXT) {
    return false;
  }

  // check G_ZEXT input register is 5bits or more and scalar type
  Register Src = AmtDef->getOperand(1).getReg();
  LLT SrcTy = MRI.getType(Src);
  if (SrcTy.isVector()) {
    return false;
  }
  if (SrcTy.getScalarSizeInBits() < Log2_32(BW)) {
    return false;
  }

  // proceed if tests have passed
  MatchInfo = Src;
  return true;
}

void applyRemoveZextWhenLoweringShl(MachineInstr &MI, MachineRegisterInfo &MRI,
                                    MachineIRBuilder &B, Register &MatchInfo) {
  Register OriginalShiftAmt = MI.getOperand(2).getReg();
  LLT OriginalShiftTy = MRI.getType(OriginalShiftAmt);
  LLT SrcTy = MRI.getType(MatchInfo);

  Register NewShiftAmt = MatchInfo;
  // if shift amount is zero extended, replace it with new shift amount
  if (OriginalShiftTy != SrcTy) {
    B.setInstrAndDebugLoc(MI);
    NewShiftAmt = B.buildAnyExt(OriginalShiftTy, MatchInfo).getReg(0);
  }

  MI.getOperand(2).setReg(NewShiftAmt);
}
```

## llvm/test/CodeGen/AArch64/GlobalISel/combine-zext-shl.ll

```llvm
; RUN: llc -mtriple=aarch64-- -global-isel -O0 -o - %s | FileCheck %s

define i8 @test_shl_i8(i8 %a, i8 %b) {
  %r = shl i8 %a, %b
  ret i8 %r
}

define i16 @test_shl_i16(i16 %a, i16 %b) {
  %r = shl i16 %a, %b
  ret i16 %r
}

```


### Reference
- LSL instruction: https://developer.arm.com/documentation/ddi0602/2026-03/Base-Instructions/LSL--register---Logical-shift-left--register---an-alias-of-LSLV-?lang=en
