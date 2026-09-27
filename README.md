# Telink TC32 Processor Specification for Ghidra

This repository contains a fairly complete processor specification for the Telink TC32 architecture, used by all of
Telink's System-On-Chips. The work herein is based on Ryan Govostes' and trust1995 work and extended with various fix-ups and actual P-code implementation.

[Ghidra]: https://www.nsa.gov/resources/everyone/ghidra/

Right now decompilation is working well with several tested TC32 ELFs.


## Usage

Copy the  `Telink_TC32` repository to `Ghidra/Processors`. Restart Ghidra.
Afterwards, when importing a TC32 binary, when prompted for the binary's "Language", select the "Telink_TC32" processor.

For analysing Telink ELFs, I use the following process (using some plugins from my GhidraPlugins repo):
1. Import the binary, do not Auto-Analyse
2. Run the fix_funcnames.py plugin
3. Run the disas_symbols.py plugin
4. Run the Auto-Analysis, without call convention identification
5. Parse the register header file into the Data Type Manager (Grab the Telink SDK, redefine the REG_ADDR%X macros in register_82XX.h as integers instead of as pointers)
6. Export the register/defines values just imported from the Data Type Manager
7. Re-Import them as labels using the exptoimp.py plugin
8. Define a new memory region via the Memory Map. For the 826x SoCs, registers are at 0x00800000, length 0x8000.

At this point the decompilation should be fairly accurate and resolve references very nicely.


## Architecture Notes

The TC32 is fairly similar to 16-bit ARM4vT Thumb-I instruction set.

It looks like the ARMv4T patents expired around 2015 and TC32:
- Does not call itself ARM
- Uses distinct instruction mnemonics
- Uses distinct instruction encodings

ELF binaries for TC32 use the machine type identifier 58.

It re-implements the original Thumb-I instruction set, with slightly modified instruction encodings.
For example on ARM Thumb, "NOP" is encoded as 0x46c0 while on TC32 "tnop" is encoded as 0x06c0. In both cases, it is an
alias for "mov r8, r8".

A few instructions like "SWI" (conditional jump with condition code 0xf) are omitted / unsupported. There is rudimentary
mention of "tserv" in the toolchain, but it is not assembled/disassembled consistently and in testing behaves similar to a "never"
condition code.

A few instructions are specific to TC32 to enable thumb-only operation:
- treti  "bx lr and restore CPSR from SPSR"
- tmcsr  "mov CPSR, rn"
- tmrcs  "mov rn, CPSR"
- tmssr  "mov SPSR, rn"
- tmrss  "mov rn, SPSR"

## Opcode list

```
TC32 encoding             TC32               ARM Thumb-I analogue    ARM Thumb encoding
-----------------------   ----------------   ----------------------  ------------------------
000000 0000 sss ddd       tands   rd,rs      AND rd,rs               010000 0000 sss ddd
000000 0001 sss ddd       txors   rd,rs      EOR rd,rs               010000 0001 sss ddd
000000 0010 sss ddd       tshftls rd,rs      LSL rd,rs               010000 0010 sss ddd
000000 0011 sss ddd       tshftrs rd,rs      LSR rd,rs               010000 0011 sss ddd
000000 0100 sss ddd       tasrs   rd,rs      ASR rd,rs               010000 0100 sss ddd
000000 0101 sss ddd       taddcs  rd,rs      ADC rd,rs               010000 0101 sss ddd
000000 0110 sss ddd       tsubcs  rd,rs      SBC rd,rs               010000 0110 sss ddd
000000 0111 sss ddd       trotrs  rd,rs      ROR rd,rs               010000 0111 sss ddd
000000 1000 sss ddd       tnand   rd,rs      TST rd,rs               010000 1000 sss ddd
000000 1001 sss ddd       tnegs   rd,rs      NEG rd,rs               010000 1001 sss ddd
000000 1010 sss ddd       tcmp    rd,rs      CMP rd,rs               010000 1010 sss ddd
000000 1011 sss ddd       tcmpn   rd,rs      CMN rd,rs               010000 1011 sss ddd
000000 1100 sss ddd       tors    rd,rs      ORR rd,rs               010000 1100 sss ddd
000000 1101 sss ddd       tmuls   rd,rs      MUL rd,rs               010000 1101 sss ddd
000000 1110 sss ddd       tbclrs  rd,rs      BIC rd,rs               010000 1110 sss ddd
000000 1111 sss ddd       tmovns  rd,rs      MVN rd,rs               010000 1111 sss ddd
00000100 ds sss ddd       tadd rd,rs         ADD rd,rs               01000100 ds sss ddd
00000101 ds sss ddd       tcmp rd,rs         CMP rd,rs               01000101 ds sss ddd
00000110 11 000 000       tnop               NOP                     mov r8, r8
00000110 ds sss ddd       tmov rd,rs         MOV rd,rs               01000110 ds sss ddd
000001110 mmmm 000        tjex rm            BX rm                   010001110 mmmm 000
                          not implemented    BLX rm (ARMv5T)         010001111 mmmm 000
00001 ddd iiiiiiii        tloadr [pc,#imm8]  LDR [pc,#imm8]          01001 ddd iiiiiiii
0001000 ooo bbb ddd       tstorer [r,r]      STR [r,r]               0101000 ooo bbb ddd
0001001 ooo bbb ddd       tstorerh[r,r]      STRH [r,r]              0101001 ooo bbb ddd
0001010 ooo bbb ddd       tstorerb [r,r]     STRB [r,r]              0101010 ooo bbb ddd
0001011 ooo bbb ddd       tloadrsb [r,r]     LDRSB [r,r]             0101011 ooo bbb ddd
0001100 ooo bbb ddd       tloadr [r,r]       LDR [r,r]               0101100 ooo bbb ddd
0001101 ooo bbb ddd       tloadrh [r,r]      LDRH [r,r]              0101101 ooo bbb ddd
0001110 ooo bbb ddd       tloadrb [r,r]      LDRB [r,r]              0101110 ooo bbb ddd
0001111 ooo bbb ddd       tloadrsh [r,r]     LDRSH [r,r]             0101111 ooo bbb ddd
00100 ddd iiiiiiii        tstorerh[r,#i]     STRH [r,#i]             10000 ddd iiiiiiii
00101 ddd iiiiiiii        tloadrh [r,#i]     LDRH [r,#i]             10001 ddd iiiiiiii
00110 ddd iiiiiiii        tstorer [sp,#i]    STR [sp,#i]             10010 ddd iiiiiiii
00111 ddd iiiiiiii        tloadr [sp,#i]     LDR [sp,#i]             10011 ddd iiiiiiii
01000 ddd iiiiiiii        tstorerb[r,#i]     STRB [r,#i]             01110 ddd iiiiiiii
01001 ddd iiiiiiii        tloadrb [r,#i]     LDRB [r,#i]             01111 ddd iiiiiiii
01010 ddd iiiiiiii        tstorer [r,#i]     STR [r,#i]              01100 ddd iiiiiiii
01011 ddd iiiiiiii        tloadr [r,#i]      LDR [r,#i]              01101 ddd iiiiiiii
011000000 iiiiiii         tadd sp,#imm7      ADD sp,#imm7            10110000 iiiiiii
011000001 iiiiiii         tsub sp,#imm7      SUB sp,#imm7            10110001 iiiiiii
01100100 rrrrrrrr         tpush {...}        PUSH {...}              10110100 rrrrrrrr
01100101 rrrrrrrr         tpush {...,lr}     PUSH {...,lr}           10110101 rrrrrrrr
01101001 xxxxxxxx         treti              Restore CPSR from SPSR + POP {pc}
01101011 11000 sss        tmcsr              MSR CPSR, Rs            not thumb in ARMv4
01101011 11001 ddd        tmrcs              MRS Rd, CPSR            not thumb in ARMv4
01101011 11010 sss        tmssr              MSR SPSR, Rs            not thumb in ARMv4
01101011 11011 ddd        tmrss              MRS Rd, SPSR            not thumb in ARMv4
01101100 rrrrrrrr         tpop {...}         POP {...}               10111100 rrrrrrrr
01101101 rrrrrrrr         tpop {...,pc}      POP {...,pc}            10111101 rrrrrrrr
01110 ddd iiiiiiii        tadd rd,pc,#imm    ADD rd,pc,#imm          10100 ddd iiiiiiii
01111 ddd iiiiiiii        tadd rd,sp,#imm    ADD rd,sp,#imm          10101 ddd iiiiiiii
10000 xxxxxxxxxxx         tj.n               B.n                     11100 xxxxxxxxxxx
10010/10011 xxxxxxxxxxx   tjl                BL                      11110/11111 ...  [32-bit]
10100 ddd iiiiiiii        tmovs r,#imm       MOVS r,#imm             00100 ddd iiiiiiii
10101 ddd iiiiiiii        tcmp r,#imm        CMP r,#imm              00101 ddd iiiiiiii
10110 ddd iiiiiiii        tadds r,#imm       ADDS r,#imm             00110 ddd iiiiiiii
10111 ddd iiiiiiii        tsubs r,#imm       SUBS r,#imm             00111 ddd iiiiiiii
1100 cccc xxxxxxxx        tj<cond>.n         B<cond>.n               1101 cond xxxxxxxx
11010 bbb rrrrrrrr        tstorem            STMIA                   11000 bbb rrrrrrrr
11011 bbb rrrrrrrr        tloadm             LDMIA                   11001 bbb rrrrrrrr
11100 iiiii sss ddd       tasrs rd,rs,#imm5  ASR rd,rs,#imm5         00010 iiiii sss ddd
1110100 nnn sss ddd       tadds rd,rs,rn     ADDS rd,rs,rn           0001100 nnn sss ddd
1110101 nnn sss ddd       tsubs rd,rs,rn     SUBS rd,rs,rn           0001101 nnn sss ddd
1110110 iii sss ddd       tadds rd,rs,#imm3  ADDS rd,rs,#imm3        0001110 iii sss ddd
1110111 iii sss ddd       tsubs rd,rs,#imm3  SUBS rd,rs,#imm3        0001111 iii sss ddd
11110 iiiii sss ddd       tshftls rd,rs,#i5  LSL rd,rs,#imm5         00000 iiiii sss ddd
11111 iiiii sss ddd       tshftrs rd,rs,#i5  LSR rd,rs,#imm5         00001 iiiii sss ddd
```

## Development Notes

As said, I used RGov's work as a basis and continued from there.
The Telink SDK contains the binutils readelf and objdump, Ryan reverse engineered the compiled objdump and found it mostly used the Thumb disassembler from ['arm-dis.c']. The opcode masks and values and the assembler format strings for each instruction were extracted. 

[arm-dis.c]: http://sourceware.org/git/gitweb.cgi?p=binutils-gdb.git;a=blob;f=opcodes/arm-dis.c;hb=HEAD

He then made the `generate_sleigh.py` script which uses the extracted opcode table to create sleigh instruction symbols and added some sensible defaults for register, RAM definitions and so on.

From here I made several improvements:
1. Fixed incorrect jump offsets
2. Defined register lists for push, pop, store, load commands ( for example tpush {r1, r2, lr} )
3. Added the 32-bit jump and link instruction
4. Added calling conventions and full P-Code implementation of the instruction set, to support the decompiler engine.

Most of the work was done by using the arm-dis.c file and the Ghidra Thumb processor specification, making adjustments when needed.
