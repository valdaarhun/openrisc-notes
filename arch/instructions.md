## Instruction sets

- ORBIS32/64
- ORFPX32/64
- ORFPX64A32
- ORVDX64

## Instruction classes

- Class I - required in implementation
- Class II - optional

## KProbes for ORBIS32

The following instructions might require single-stepping emulation instead of SSOL.

- l.adrp    0x2  (first 6 bits)
- l.bf      0x4  (first 6 bits)
- l.bnf     0x3  (first 6 bits)
- l.j       0x0  (first 6 bits)
- l.jal     0x1  (first 6 bits)
- l.jalr    0x12 (first 6 bits)
- l.jr      0x11 (first 6 bits)


Some instructions that should not be supported for KProbes

- l.csync   0x23000000
- l.msync   0x22000000
- l.psync   0x22800000

- l.lwa     0x1b (first 6 bits)
- l.swa     0x33 (first 6 bits)

- l.macrc   0x6 (first 6 bits) and 0x10000 (last 17 bits)

- l.mtspr   0x30 (first 6 bits)
- l.mfspr   0x2d (first 6 bits)

- l.rfe     0x9 (first 6 bits)
- l.sys     0x2000 (first 16 bits)
- l.trap    0x2100 (first 16 bits)
