


`MachineRegisterInfo`: save 
- A->B: If B uses A, 

## selectMachineFunction

```cpp
bool InstructionSelect::selectMachineFunction(MachineFunction &MF) {
  LLVM_DEBUG(dbgs() << "Selecting function: " << MF.getName() << '\n');
  assert(ISel && "Cannot work without InstructionSelector");

  CodeGenCoverage CoverageInfo;
  ISel->setupMF(MF, VT, &CoverageInfo, PSI, BFI);

  // An optimization remark emitter. Used to report failures.
  MachineOptimizationRemarkEmitter MORE(MF, /*MBFI=*/nullptr);
  ISel->MORE = &MORE;

  // FIXME: There are many other MF/MFI fields we need to initialize.

  MachineRegisterInfo &MRI = MF.getRegInfo();
#ifndef NDEBUG
  // Check that our input is fully legal: we require the function to have the
  // Legalized property, so it should be.
  // FIXME: This should be in the MachineVerifier, as the RegBankSelected
  // property check already is.
  if (!DisableGISelLegalityCheck)
    if (const MachineInstr *MI = machineFunctionIsIllegal(MF)) {
      reportGISelFailure(MF, MORE, "gisel-select", "instruction is not legal",
                         *MI);
      return false;
    }
#endif
  // Keep track of selected blocks, so we can delete unreachable ones later.
  DenseSet<MachineBasicBlock *> SelectedBlocks;

  {
    // Observe IR insertions and removals during selection.
    // We only install a MachineFunction::Delegate instead of a
    // GISelChangeObserver, because we do not want notifications about changed
    // instructions. This prevents significant compile-time regressions from
    // e.g. constrainOperandRegClass().
    GISelObserverWrapper AllObservers;
    MIIteratorMaintainer MIIMaintainer;
    AllObservers.addObserver(&MIIMaintainer);
    RAIIDelegateInstaller DelInstaller(MF, &AllObservers);
    ISel->AllObservers = &AllObservers;

    // Use a DFS stack to maintain post-order traversal. Each entry holds a
    // block and an iterator over its successors that we have yet to visit.
    // A block is selected only when all its successors have been selected
    // (i.e. when the successor iterator is exhausted), preserving post-order.
    // If selecting a block introduces new blocks, we snapshot the vreg
    // register-class/bank state beforehand so we can roll it back, re-insert
    // the new blocks into the DFS stack in the right position, and re-select
    // the original block with correct successor information.
    using SuccIt = MachineBasicBlock::succ_iterator;
    SmallVector<std::pair<MachineBasicBlock *, SuccIt>, 16> DFSStack;
    DenseSet<MachineBasicBlock *> Visited;

    auto PushBlock = [&](MachineBasicBlock *MBB) {
      if (Visited.insert(MBB).second)
        DFSStack.emplace_back(MBB, MBB->succ_begin());
    };
    PushBlock(&MF.front());

    while (!DFSStack.empty()) {
      auto &[MBB, SuccI] = DFSStack.back();

      // If there are unvisited successors, descend into them first.
      if (SuccI != MBB->succ_end()) {
        MachineBasicBlock *Succ = *SuccI++;
        PushBlock(Succ);
        continue;
      }

      // All successors visited — this block is next in post-order.
      // Snapshot the RegClassOrRegBank of every vreg defined or used in MBB
      // so we can restore state if new blocks are introduced during selection.
      DenseMap<Register, RegClassOrRegBank> Snapshot;
      for (const MachineInstr &MI : *MBB)
        for (const MachineOperand &MO : MI.operands())
          if (MO.isReg() && MO.getReg().isVirtual())
            Snapshot.try_emplace(MO.getReg(),
                                 MRI.getRegClassOrRegBank(MO.getReg()));

      const size_t NumBlocksBefore = MF.size();

      ISel->CurMBB = MBB;
      SelectedBlocks.insert(MBB);

      // Select instructions in reverse block order.
      MIIMaintainer.MII = MBB->rbegin();
      for (auto End = MBB->rend(); MIIMaintainer.MII != End;) {
        MachineInstr &MI = *MIIMaintainer.MII;
        ++MIIMaintainer.MII;

        LLVM_DEBUG(dbgs() << "\nSelect:  " << MI);
        if (!selectInstr(MI)) {
          LLVM_DEBUG(dbgs() << "Selection failed!\n";
                     MIIMaintainer.reportFullyCreatedInstrs());
          reportGISelFailure(MF, MORE, "gisel-select", "cannot select", MI);
          return false;
        }
        LLVM_DEBUG(MIIMaintainer.reportFullyCreatedInstrs());
      }

      // Check whether new blocks were introduced during selection.
      if (MF.size() != NumBlocksBefore) {
        LLVM_DEBUG(dbgs() << "New blocks introduced while selecting BB#"
                          << MBB->getNumber() << "; re-queuing.\n");

        // Restore vreg attributes so the re-selection sees a clean state.
        for (auto &[Reg, RCOrRB] : Snapshot)
          MRI.setRegClassOrRegBank(Reg, RCOrRB);

        // Mark MBB as unselected so it will be re-selected after its new
        // successors have been processed.
        SelectedBlocks.erase(MBB);

        // Pop MBB from the DFS stack and re-push it together with any new
        // successors so the DFS naturally visits them first.
        DFSStack.pop_back();
        // Re-push MBB with a fresh successor iterator; PushBlock will skip
        // already-visited successors via the Visited set.
        Visited.erase(MBB);
        PushBlock(MBB);
        continue;
      }

      DFSStack.pop_back();
    }
  }

  for (MachineBasicBlock &MBB : MF) {
    if (MBB.empty())
      continue;

    if (!SelectedBlocks.contains(&MBB)) {
      // This is an unreachable block and therefore hasn't been selected, since
      // the main selection loop above uses a postorder block traversal.
      // We delete all the instructions in this block since it's unreachable.
      MBB.clear();
      // Don't delete the block in case the block has it's address taken or is
      // still being referenced by a phi somewhere.
      continue;
    }
    // Try to find redundant copies b/w vregs of the same register class.
    for (auto MII = MBB.rbegin(), End = MBB.rend(); MII != End;) {
      MachineInstr &MI = *MII;
      ++MII;

      if (MI.getOpcode() != TargetOpcode::COPY)
        continue;
      Register SrcReg = MI.getOperand(1).getReg();
      Register DstReg = MI.getOperand(0).getReg();
      if (SrcReg.isVirtual() && DstReg.isVirtual()) {
        auto SrcRC = MRI.getRegClass(SrcReg);
        auto DstRC = MRI.getRegClass(DstReg);
        if (SrcRC == DstRC) {
          MRI.replaceRegWith(DstReg, SrcReg);
          MI.eraseFromParent();
        }
      }
    }
  }

#ifndef NDEBUG
  const TargetRegisterInfo &TRI = *MF.getSubtarget().getRegisterInfo();
  // Now that selection is complete, there are no more generic vregs.  Verify
  // that the size of the now-constrained vreg is unchanged and that it has a
  // register class.
  for (unsigned I = 0, E = MRI.getNumVirtRegs(); I != E; ++I) {
    Register VReg = Register::index2VirtReg(I);

    MachineInstr *MI = nullptr;
    if (!MRI.def_empty(VReg))
      MI = &*MRI.def_instr_begin(VReg);
    else if (!MRI.use_empty(VReg)) {
      MI = &*MRI.use_instr_begin(VReg);
      // Debug value instruction is permitted to use undefined vregs.
      if (MI->isDebugValue())
        continue;
    }
    if (!MI)
      continue;

    const TargetRegisterClass *RC = MRI.getRegClassOrNull(VReg);
    if (!RC) {
      reportGISelFailure(MF, MORE, "gisel-select",
                         "VReg has no regclass after selection", *MI);
      return false;
    }

    const LLT Ty = MRI.getType(VReg);
    if (Ty.isValid() &&
        TypeSize::isKnownGT(Ty.getSizeInBits(), TRI.getRegSizeInBits(*RC))) {
      reportGISelFailure(
          MF, MORE, "gisel-select",
          "VReg's low-level type and register class have different sizes", *MI);
      return false;
    }
  }

#endif

  if (!DebugCounter::shouldExecute(GlobalISelCounter)) {
    dbgs() << "Falling back for function " << MF.getName() << "\n";
    MF.getProperties().setFailedISel();
    return false;
  }

  // Determine if there are any calls in this machine function. Ported from
  // SelectionDAG.
  MachineFrameInfo &MFI = MF.getFrameInfo();
  for (const auto &MBB : MF) {
    if (MFI.hasCalls() && MF.hasInlineAsm())
      break;

    for (const auto &MI : MBB) {
      if ((MI.isCall() && !MI.isReturn()) || MI.isStackAligningInlineAsm())
        MFI.setHasCalls(true);
      if (MI.isInlineAsm())
        MF.setHasInlineAsm(true);
    }
  }

  // FIXME: FinalizeISel pass calls finalizeLowering, so it's called twice.
  auto &TLI = *MF.getSubtarget().getTargetLowering();
  TLI.finalizeLowering(MF);

  LLVM_DEBUG({
    dbgs() << "Rules covered by selecting function: " << MF.getName() << ":";
    for (auto RuleID : CoverageInfo.covered())
      dbgs() << " id" << RuleID;
    dbgs() << "\n\n";
  });
  CoverageInfo.emit(CoveragePrefix,
                    TLI.getTargetMachine().getTarget().getBackendName());

  // If we successfully selected the function nothing is going to use the vreg
  // types after us (otherwise MIRPrinter would need them). Make sure the types
  // disappear.
  MRI.clearVirtRegTypes();

  // FIXME: Should we accurately track changes?
  return true;
}

```
