# Make gMIR->MIR Instruction Selection to Process newly created block while Selection

## lib/CodeGen/GlobalISel/InstructionSelect.cpp

```cpp
#include "llvm/ADT/DenseMap.h"
#include "llvm/ADT/DenseSet.h"
...

    GISelObserverWrapper AllObservers;
    MIIteratorMaintainer MIIMaintainer;
    AllObservers.addObserver(&MIIMaintainer);
    RAIIDelegateInstaller DelInstaller(MF, &AllObservers);
    ISel->AllObservers = &AllObservers;

    SmallVector<
        std::pair<MachineBasicBlock *, MachineBasicBlock::succ_iterator>>
        BlockSuccessorList;
    DenseSet<MachineBasicBlock *> VisitedBlocks;

    auto AppendBlockSuccessor = [&](MachineBasicBlock *Block) {
      if (!VisitedBlocks.contains(Block)) {
        VisitedBlocks.insert(Block);
        BlockSuccessorList.emplace_back(Block, Block->succ_begin());
      }
    };
    AppendBlockSuccessor(&MF.front());

    while (!BlockSuccessorList.empty()) {
      MachineBasicBlock *&CurrentBlock = BlockSuccessorList.back().first;
      MachineBasicBlock::succ_iterator &CurrentBlockSuccessorIterator =
          BlockSuccessorList.back().second;

      // If there are unvisited successors, descend into them first.
      if (CurrentBlockSuccessorIterator != CurrentBlock->succ_end()) {
        MachineBasicBlock *CurrentBlockSuccessor =
            *CurrentBlockSuccessorIterator++;
        AppendBlockSuccessor(CurrentBlockSuccessor);
        continue;
      }

      // All successors visited — select this block.
      // Snapshot vreg register-class/bank state before selection so we can
      // restore it if new blocks are introduced during selection.
      DenseMap<Register, RegClassOrRegBank> VirtualRegisterCache;
      for (const MachineInstr &Instruction : *CurrentBlock)
        for (const MachineOperand &Operand : Instruction.operands())
          if (Operand.isReg() && Operand.getReg().isVirtual())
            VirtualRegisterCache.try_emplace(
                Operand.getReg(), MRI.getRegClassOrRegBank(Operand.getReg()));

      const size_t BlockCountBeforeSelection = MF.size();

      ISel->CurMBB = CurrentBlock;
      SelectedBlocks.insert(CurrentBlock);

      MIIMaintainer.MII = CurrentBlock->rbegin();
      for (auto End = CurrentBlock->rend(); MIIMaintainer.MII != End;) {
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

      if (MF.size() > BlockCountBeforeSelection) {
        for (auto &VirtualRegisterEntry : VirtualRegisterCache)
          MRI.setRegClassOrRegBank(VirtualRegisterEntry.first,
                                   VirtualRegisterEntry.second);
        SelectedBlocks.erase(CurrentBlock);
        BlockSuccessorList.pop_back();
        VisitedBlocks.erase(CurrentBlock);
        AppendBlockSuccessor(CurrentBlock);
        continue;
      }
      BlockSuccessorList.pop_back();
    }
  }

  for (MachineBasicBlock &MBB : MF) {
    if (MBB.empty())
      continue;
```
