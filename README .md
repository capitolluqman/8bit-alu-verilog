# 8-bit ALU — RTL Design \& Verification

A parameterized 8-bit Arithmetic Logic Unit implemented in SystemVerilog, with a
self-checking testbench that combines directed tests, constrained-random
regression, concurrent SystemVerilog Assertions (SVA), and functional coverage.

## Why this project

Built to apply RTL design and verification fundamentals — Verilog/SystemVerilog,
testbench development, SVA, and functional coverage — to a small but complete
design, in preparation for RTL design internship applications.

## Design overview

`alu.sv` implements an 8-bit ALU supporting 8 operations, selected by a 3-bit
opcode:

|Opcode|Operation|Notes|
|-|-|-|
|000|ADD|8-bit unsigned add, with carry-out|
|001|SUB|8-bit unsigned subtract, with borrow-out|
|010|AND|Bitwise AND|
|011|OR|Bitwise OR|
|100|XOR|Bitwise XOR|
|101|NOT|Bitwise NOT of `a`|
|110|SHL|Logical shift-left of `a` by 1 bit|
|111|SHR|Logical shift-right of `a` by 1 bit|

**Flags:**

* `zero` — asserted when `result == 0`
* `carry\_out` — carry out of MSB (ADD) / borrow indicator (SUB)
* `overflow` — signed 2's-complement overflow, meaningful only for ADD/SUB

The design is fully combinational (`always\_comb`), with default assignments
at the top of the block to avoid unintended latches — this is a standard RTL
coding practice: every signal driven inside the block is given a safe
default value first, so no branch of the `case` statement can accidentally
leave a signal undriven.

## Verification approach

`alu\_tb.sv` verifies the design using:

* **Reference model** — a behavioral golden model (`ref\_result` function)
used as the scoreboard's expected-value source. It computes each result
using plain operators, independently of the DUT's internal logic.
* **Directed tests** — one test per opcode plus known edge cases: unsigned
carry without signed overflow, signed overflow on ADD and SUB, and
zero-result cases.
* **Constrained-random regression** — 50 randomized `(a, b, opcode)`
combinations, each checked against the reference model.
* **Immediate SVA (SystemVerilog Assertions)** — `assert` statements
checked right after each test case settles, confirming the `zero` flag
always matches `result == 0`, and that `NOT` never raises `carry\_out`
or `overflow`.
* **Functional coverage** — tracked manually with plain arrays/counters
rather than a SystemVerilog `covergroup`. Icarus Verilog does not support
the `covergroup` construct, so this project tracks "did we exercise every
opcode, and did we see each flag go both 0 and 1" using simple variables
instead — same goal (know whether your tests were thorough), simulator-
portable implementation.

Because the ALU is purely combinational (no clock, no internal state), the
testbench has no clock either — each test simply drives inputs, waits a
short delay for the logic to settle, then checks the outputs.

## How to run

Verified to run on **EDA Playground with Icarus Verilog 12.0** (compile
options: `-Wall -g2012`). This is the free, no-account-required path on EDA
Playground - Riviera-PRO/Questa require a registered account.

### Option A — EDA Playground (no install, recommended)

1. Go to https://www.edaplayground.com
2. Select **Icarus Verilog** from the Tools \& Simulators dropdown.
3. Paste `alu.sv` into the "Design" pane and `alu\_tb.sv` into the
"Testbench" pane.
4. Make sure the compile options include `-g2012` (needed for
`always\_comb`, `logic`, and other SystemVerilog-2012 syntax).
5. Click Run.

### Option B — Xilinx Vivado (local)

1. Create a new RTL project, add `alu.sv` as a design source and
`alu\_tb.sv` as a simulation source.
2. Set `alu\_tb` as the top module for simulation.
3. Run Behavioral Simulation.
4. View the waveform in the Vivado waveform viewer; check the Tcl console
for the `$display` pass/fail summary and coverage percentage.

### Option C — Icarus Verilog (local)

```bash
iverilog -Wall -g2012 -o sim alu.sv alu\_tb.sv
vvp sim
```

### A note on portability

The testbench intentionally avoids two constructs that are known to cause
trouble on Icarus Verilog specifically:

* The `string` data type in task port lists (uses plain integer test IDs
instead)
* The `covergroup` construct, which Icarus does not support at all
(coverage is tracked manually with plain variables instead)

It also sticks to plain ASCII throughout, since non-ASCII characters
(smart quotes, em-dashes) in source files have also been known to cause
confusing parser errors. Good habits for simulator-portable RTL/testbench
code generally.

## Results

Verified on Icarus Verilog 12.0 (EDA Playground), 8 August 2026:

* Tests run: **66** (16 directed edge-case tests + 50 constrained-random tests)
* Errors: **0**
* Opcode coverage: **8 / 8** operations exercised
* Flag coverage: `zero`, `carry\_out`, and `overflow` each observed in both
the 0 and 1 states
* Overall functional coverage: **100.00% (14 / 14 bins)**
* Result: **ALL TESTS PASSED**

Console output:

```
=== 8-bit ALU Verification Start ===
=== 8-bit ALU Verification Complete ===
Tests run   : 66
Errors      : 0
Opcode coverage : 8 / 8 opcodes exercised
Flag coverage   : zero(0/1)=1/1  carry(0/1)=1/1  overflow(0/1)=1/1
Overall functional coverage : 100.00% (14 / 14 bins)
RESULT: ALL TESTS PASSED
```

Note: Icarus reports a "sorry: constant selects in always\_\* processes are
not currently supported" message for the bit-select operations in the ALU's
`always\_comb` block. This is a known Icarus optimization limitation, not a
correctness issue — it falls back to simulating the full signal instead of
optimizing the slice, and does not affect simulation results.

## Possible extensions

* Add a formal verification wrapper (e.g., SymbiYosys) to formally prove the
zero-flag and overflow properties instead of only simulating them.
* Parameterize beyond 8-bit and re-run coverage to confirm scalability.
* Add pipelining and a simple FSM-based control wrapper around the ALU.

## Author

Luqman Shariff — B.Tech VLSI Design, Presidency University

