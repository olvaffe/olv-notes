RISC-V
======

## Overview

- <https://github.com/riscv/riscv-isa-manual>
  - `RV32I` is for 32-bit microcontrollers
    - `RV32E` is a reduced subset
  - `RV64I` is for 64-bit application processors
    - `RV64E` is a reduced subset
- <https://github.com/riscv/riscv-profiles>
  - `RVM` is for 32-bit microcontrollers
  - `RVA` is for 64-bit application processors
    - `RVB` is a reduced subset

## `RV32I`

- registers
  - one `pc` register
  - one `x0` register that is hardwired to 0
  - 31 `x1-x31` general purpose registers
  - all registers are 32-bit
- instruction formats
  - R: funct7 | rs2 | rs1 | func3 | rd | opcode
    - opcode has 7 bits and is always `0110011` for R-format instructions
    - rd and rs1/rs2 are dst and src registers, 5-bit each
    - func3/func7 has 3/7 bits each.
    - ADD/SUB/SLL/SLT/SLTU/XOR/SRL/SRA/OR/AND
  - I: imm[11:0] | rs1 | func3 | rd | opcode
    - func7 and rs2 are replaced by a 12-bit signed imm
    - ADDI/SLT1/SLTIU/XORI/ORI/ANDI/SLLI/SRLI/SRAI
      - opcode is always `0010011`
    - LB/LH/LW/LBU/LHU for loads
      - opcode is always `0000011`
      - imm holds the offset from rs1
    - JALR for unconditional jump
      - rd is set to pc+4 before jump (return address)
      - pc is set to rs1+immediate
  - S: imm[11:5] | rs2 | rs1 | func3 | imm[4:0] | opcode
    - for stores; no rd needed
    - SB/SH/SW
      - opcode is always `0100011`
      - rs1/imm is the addr/offset
      - rs2 is the value
  - B: imm[12] | imm[10:5] | rs2 | rs1 | func3 | imm[4:1] | imm[11] | opcode
    - for branching; imm is offset from pc
    - imm[0] is assumed to be 0 (2-byte increments)
    - BEQ/BNE/BLT/BGE/BLTU/BGEU
  - U: imm[31:12] | rd | opcode
    - operates the upper 20 bits of immediates
    - LUI: set the higher 20 bits and clear the lower 12 bits
    - AUIPC: set rd to pc+imm[31:12], used to get current pc mostly
  - J: imm[20] | imm[10:1] | imm[11] | imm[19:12] | rd | opcode
    - for unconditional jump; similar to B
    - rd is set to pc+4 before jump, as the return address
