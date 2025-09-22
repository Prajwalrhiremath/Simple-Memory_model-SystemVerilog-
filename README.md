# memory\_model\_fixed — README

## Overview

`memory_model_fixed` is a compact, flexible SystemVerilog memory model implemented as a class and accompanied by a demonstration testbench. It provides a 32-bit word-addressable dynamic memory array with multiple addressing modes (auto, word-index, byte-address) and utilities for reading, writing, memory dumps, and loading initialization data from a hex file.

This model is intended for behavioral simulation (testbenches, bus models, simple memory backends) and is **not** synthesizable.

---

## Features

* 32-bit wide dynamic memory (`mem_array`) with configurable size (words).
* Addressing modes (`addr_mode_t`):

  * `ADDR_AUTO` — tries word-index first, then byte-address (`addr/4`).
  * `ADDR_WORD` — interprets inputs as word indices (0..mem\_size-1).
  * `ADDR_BYTE` — interprets inputs as byte addresses (0,4,8,... -> index `addr/4`).
* Optional handling of unaligned byte addresses (warning or error) via `allow_unaligned` flag.
* Per-call override capability (explicit override enable) to force word/byte interpretation.
* `read` / `write` API with safe bounds checking and helpful messages (`$display`, `$warning`, `$error`).
* `display_memory(start_index, num_words)` for human-readable memory dumps.
* `load_from_hex_file(filename)` to initialize memory from a text file containing hex words (one per line).

---

## Files

* `memory_model_fixed2.sv` — main SystemVerilog source containing the `memory_model` class and the example `testbench`.

(If you prefer the earlier file names, you'll also see `memory_model_fixed.sv` in prior iterations.)

---

## Quick usage

1. Include `memory_model_fixed2.sv` in your simulator file list.
2. Run simulation (examples below).
3. The provided `testbench` demonstrates the typical API and prints memory operations.

### Instantiate the class

```systemverilog
// create a 1024-word memory, AUTO addressing, allow unaligned addresses
memory_model mem = new(1024, ADDR_AUTO, 1);
```

### Basic read/write (no override)

```systemverilog
mem.write(0, 32'h1234_5678, ADDR_AUTO, 0); // writes at word index 0
bit [31:0] data = mem.read(0, ADDR_AUTO, 0);
```

### Byte address (AUTO will try byte mapping if needed)

```systemverilog
mem.write(2000, 32'hFFFF_FFFF, ADDR_AUTO, 0); // 2000 >> 2 = 500 -> writes to word index 500
```

### Force byte interpretation for a call

```systemverilog
mem.write(4, 32'hDEAD_BEEF, ADDR_BYTE, 1); // treat '4' as byte address -> index 1
```

### Display memory

```systemverilog
mem.display_memory(0, 16); // prints first 16 words
```

### Load from hex file (optional)

* Create a file `init_mem.hex` with one hex word per line (with or without `0x` prefix).
* Call:

```systemverilog
int unsigned loaded = mem.load_from_hex_file("init_mem.hex");
```

---

## Running with common simulators

### Questa/ModelSim (example)

```bash
# compile
vlog memory_model_fixed2.sv
# simulate
vsim -c work.testbench -do "run -all; exit"
```

### VCS (example)

```bash
vcs -sverilog memory_model_fixed2.sv -o simv
./simv +vcs+finish+1
```

### Icarus/iverilog

> Note: Icarus does not fully support SystemVerilog class features. Use a simulator that implements SystemVerilog class semantics (Questa, VCS, Riviera-PRO, etc.).

---

## API summary

* `function new(int unsigned size_words = 256, addr_mode_t addr_mode = ADDR_AUTO, bit allow_unaligned_byte = 0)` — constructor.
* `task write(bit [31:0] addr, bit [31:0] data, input addr_mode_t override_mode, input bit override_enable)` — write a 32-bit word. If `override_enable==1` then `override_mode` determines interpretation for that call; otherwise the class-level `mode` is used.
* `function bit [31:0] read(bit [31:0] addr, input addr_mode_t override_mode, input bit override_enable)` — read a 32-bit word. Same override semantics as `write`.
* `task display_memory(int unsigned start_index = 0, int unsigned num_words = 16)` — pretty-print a block of memory.
* `function int unsigned load_from_hex_file(string filename)` — load hex words (returns number loaded).

Return and error behavior

* On out-of-bounds reads/writes the model prints `$error` and `read` returns `32'hdead_beef` on failures.

---

## Design notes & rationale

* The model treats memory as a dynamic array of 32-bit words because many testbenches prefer word-level accesses; byte-addressing is supported for convenience.
* `ADDR_AUTO` is friendly for mixed usage (small values interpreted as indices, larger values as byte addresses).
* `override_enable` boolean is used instead of passing `'x` for enums to avoid simulator warnings and make intent explicit.
* Initialization and all declarations are placed carefully to avoid "declarations after statements" errors with strict simulators.

---

## Troubleshooting

* **`MEM_WRITE_ERROR`**: you attempted to write a value that is out of bounds for the current `mem_size` **under the chosen interpretation** (word vs byte). Check whether your address is a word index or a byte address and either change the address, change `mem_size`, or use override mode accordingly.
* **Enum `'x` warnings**: resolved in the supplied file by using `override_enable`. Don't pass `'x` for enum arguments.
* **File not found for `init_mem.hex`**: either create the file in the simulator working directory or remove the load call.
* **Simulator compatibility**: use a SystemVerilog simulator with class support (Questa, VCS, etc.). Icarus/iverilog has limited SystemVerilog class support.

---

## Extensions you might want

* Add byte-enable support (per-byte masking within a 32-bit write).
* Provide a return status (success/failure) for `write`/`read` instead of relying on `$error` messages.
* Expose both a word-index API and a byte-address API separately for clarity.
* Add an optional callback/hook for external observers on every write (helpful for testbench monitors).

---

## License & contact

Use freely in your testbenches and projects. If you want help adapting the model to your environment or adding features, mention your simulator and desired behavior.

---

*End of README.*
