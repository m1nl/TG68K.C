# TG68K.C

switchable 68K CPU-Core

The TG68K.C is an IP core for FPGAs. It can be easily switched between a 68000, a 68010 and a 68020.
There are now many projects in the retro computer area that use this core. There was a lot of feedback from this area. So the core could develop very well. Many bugs could also be eliminated.
The core does not value cycle accuracy. The core saves the FPGA resources with a good execution speed. 



Der TG68K.C ist ein IP Core fuer FPGAs. Er kann auf einfache Weise zwischen einem 68000, einem 68010 und einem 68020 umgeschaltet werden. 
Es gibt inzwischen viele Projekte im Retrocomputer Bereich die diesen Core verwenden. Aus diesem Bereich gab es viele Rückmeldungen. So konnte der Core sehr gut weiter entwickelt werden. Ebenso konnten dadurch viele Bugs beseitigt werden. 
Der Core legt keinen Wert auf Zyklusgenauigkeit. Der Core schont die FPGA Resourcen bei einer guten Ausführungsgeschwindigkeit.


## 68020 32-bit data path extension

`TG68KdotC_Kernel.vhd` in this directory contains the 32-bit data path
extracted from commit `ec1353dc1b36e1c1a91fe14786ec7e09ec2a2cf1`, without its
MMU additions, plus paired consumption of selected instruction extensions.
The package and ALU retain their existing execution logic.

Wide transfers are available only when the kernel's `CPU` input is `"11"`
(68020). The memory controller may accept each eligible transfer separately;
keeping `long_ack='0'` selects the regular 16-bit bus sequence. No `s21_en`
input is used.

### Compile-time feature flags

The kernel generic `DataPath32 : std_logic_vector(2 downto 0) := "111"`
independently enables three kinds of aligned pair transfers:

| Bit | Feature |
| --- | --- |
| 0 | Longword memory data reads and writes |
| 1 | Selected 32-bit immediate operand fetches |
| 2 | Selected absolute-long address extension fetches |

For example, `generic map (DataPath32 => "001")` enables memory data pairs
while retaining narrow immediate/address fetches. `"010"` enables immediates
only, `"100"` enables absolute addresses only, `"111"` enables all three, and
`"000"` disables all three. Other combinations enable their selected bits.
All pair transfers still require `CPU="11"` and the existing `long_ack`
handshake. Disabling an address fetch does not disable a subsequent wide data
access, and enabling an address fetch does not enable wide data accesses.

The existing `longword` phase indication is available for every setting,
including `"000"`, so legacy narrow-write combining can still use it.
With memory pairs disabled, `data_write_lo` is tied to zero.

The kernel uses a single `logic` architecture. `"000"` disables the three
pair qualifiers and retains the normal narrow transfer sequence. These are
compile-time flags, so synthesis can remove disabled feature logic. There is
no separate legacy implementation or architecture selector. Existing
instantiations retain all three features by default. Component instantiations
that set the generic must add it to their component declaration.

### Feature-flag resource check

Isolated GHDL/Yosys ECP5 synthesis with `CPU` fixed to `"11"`:

| Kernel | LUT4 | PFUMX | L6MUX21 | Flip-flops |
| --- | ---: | ---: | ---: | ---: |
| Reference commit `68007f8` | 8287 | 2043 | 700 | 954 |
| `DataPath32="000"` | 8374 | 2073 | 766 | 954 |
| `DataPath32="111"` | 8580 | 2089 | 746 | 955 |

The disabled build adds 87 LUT4s (about 1.0%) and no flip-flops; all features
add 293 LUT4s (about 3.5%) and one flip-flop. Carry, DSP and distributed-RAM
counts are unchanged. These are synthesis results, not Diamond mapping or
board timing measurements. With a switchable CPU input, the disabled build
uses 8573 LUT4s versus 8494 in the reference, with 955 flip-flops in both.

### Memory controller interface

| Port | Direction | Meaning |
| --- | --- | --- |
| `long_cand` | Output | The current access can transfer both 16-bit words together. |
| `long_ack` | Input | Accept this pair on an enabled clock edge (`clkena_in='1'`). Defaults to zero. |
| `data_in_hi` | Input | Bits 31:16 of an accepted wide read. Defaults to zero. |
| `data_write_lo` | Output | Bits 15:0 of an accepted wide write. |

The bus is big endian. For an accepted pair starting at `addr_out`:

| Transfer | Word at address + 0 | Word at address + 2 |
| --- | --- | --- |
| Read | `data_in_hi` (bits 31:16) | `data_in` (bits 15:0) |
| Write | `data_write` (bits 31:16) | `data_write_lo` (bits 15:0) |

For a narrow read, `data_in` supplies the word at the current address.
Only four-byte-aligned pairs are offered. Unaligned accesses follow the
existing narrow sequence. `clkena_in` controls progress for both narrow and
wide accesses; acknowledgement alone does not advance the core.

Use `long_cand` to identify eligible pairs. The existing `longword` output
retains its original transfer-phase meaning and need not be asserted for a
paired instruction-extension read. `busstate="00"` identifies instruction
stream reads; `"10"` and `"11"` identify data reads and writes respectively.

These ports belong to the kernel. Integrations using the supplied
`TG68K.vhd` wrapper must update its component declaration and port mapping,
and expose the new signals to the memory controller, to use wide transfers.

### Data and instruction-extension transfers

Aligned ordinary longword data reads and writes can complete in a single
accepted pair. Bitfield accesses are excluded from the ordinary-longword
qualifier. The existing data path handles accesses that are not accepted
as wide transfers.

The following instruction extensions can also be consumed as one pair:

| Instructions | Extension consumed together |
| --- | --- |
| `MOVE.L #imm,Dn`, `MOVEA.L #imm,An` | 32-bit immediate |
| `MOVE.L #imm,(An)`, `(An)+`, `-(An)`, `d16(An)`, `(abs.W/L)` | 32-bit immediate |
| `ORI.L`, `ANDI.L`, `SUBI.L`, `ADDI.L`, `EORI.L`, `CMPI.L #imm,Dn/(An)/(An)+/-(An)/d16(An)/(abs.W/L)` | 32-bit immediate |
| `ADDA.L`, `SUBA.L`, `CMPA.L #imm,An` | 32-bit immediate |
| `MOVE.B/W/L (abs.L),Dn`, `MOVEA.W/L (abs.L),An` | 32-bit source address |
| `MOVE.B/W/L (abs.L),(An)` / `(An)+` / `-(An)` / `d16(An)` / `(abs.W/L)` | 32-bit source address |
| `LEA (abs.L),An`, `PEA (abs.L)` | 32-bit address |
| `TST.B/W/L (abs.L)`, `CLR.B/W/L (abs.L)` | 32-bit address |
| `MOVE.B/W/L Dn,(abs.L)`, `MOVE.W/L An,(abs.L)` | 32-bit destination address |
| `ADD.B/W/L`, `SUB.B/W/L`, `CMP.B/W/L`, `AND.B/W/L`, `OR.B/W/L (abs.L),Dn` | 32-bit source address |
| `ADDA.W/L`, `SUBA.W/L`, `CMPA.W/L (abs.L),An` | 32-bit source address |

For every row, the extension's first word must be four-byte aligned and
the controller must acknowledge the pair. Register forms cover D0-D7 and
A0-A7 as applicable. A wide address-extension fetch does not require the
addressed data itself to be aligned; its data access is handled separately.

Pairing skips the following low-extension-word bus step. The kernel assembles
all 32 bits, preserves them through saved-data/sign-extension handling,
advances the PC and next fetch address by four, and supplies the existing
execution logic with the state reached after consuming both words normally.
Partial-register writes, MOVEA.W sign extension, arithmetic flags and
address-register flag preservation use their existing execution paths.

Each accepted extension pair saves one core step. This saving remains when
a memory controller caches the second word and could provide it without
wait states. With ordinary memory, pairing also avoids the wait states of
that second transaction. Overall speedup depends on instruction mix,
alignment, bus acceptance and memory latency.

Ordinary opcode fetches remain 16-bit; there is no instruction buffer.
Other instruction forms, including other memory destinations, JMP/JSR address
extensions and long branch displacements, retain their existing extension
fetch sequence. Byte/word immediate operands are not paired.
