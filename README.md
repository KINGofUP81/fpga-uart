# FPGA UART (Zybo)

Educational UART implementation for the Digilent Zybo board, kept in three parallel variants:

- **`uart-vhdl/`** — complete VHDL implementation with testbenches for every block.
- **`uart-systemverilog/`** — direct SystemVerilog translation of the same architecture.
- **`uart-systemverilog-mod/`** — work-in-progress extension that adds FIFO status outputs (almost empty/full, word count). It is **not currently compilable**; see [Known issues](#known-issues).

> Status: source-level review only. There are no Vivado project files or build scripts in this tree, and nothing here has been compiled or simulated as part of this documentation pass.

## Origin / attribution

The UART and FIFO RTL is adapted from the textbook examples by **Pong P. Chu**:

- *FPGA Prototyping by SystemVerilog Examples: Xilinx MicroBlaze MCS SoC Edition* (Wiley, 2018) — Chapter 7 “FIFO” (Listings 7.2, 7.7, 7.8) and the UART chapter that introduces `baud_gen`, `uart_rx`, `uart_tx`, and the `uart` wrapper.
- *FPGA Prototyping by VHDL Examples* — corresponding VHDL version of the same designs.

The Zybo constraints and top-level board wrappers were added/retargeted separately. No license file was present in the original sources.

---

## Directory layout

```text
fpga-uart/
├── uart-vhdl/
│   ├── const/zybi-master.xdc
│   ├── sim/
│   │   ├── baud_gen_tb.vhd
│   │   ├── fifo_ctrl_tb.vhd
│   │   ├── fifo_tb.vhd
│   │   ├── reg_file_tb.vhd
│   │   ├── uart_rx_tb.vhd
│   │   ├── uart_tx_tb.vhd
│   │   ├── uart_tb.vhd
│   │   └── top_uart_tb.vhd
│   └── src/
│       ├── baud_gen.vhd
│       ├── fifo.vhd
│       ├── fifo_ctrl.vhd
│       ├── reg_file.vhd
│       ├── uart.vhd
│       ├── uart_rx.vhd
│       ├── uart_tx.vhd
│       └── top_uart.vhd
├── uart-systemverilog/
│   ├── const/zybi-master.xdc
│   └── src/
│       ├── baud_gen.sv
│       ├── fifo.sv
│       ├── fifo_ctrl.sv
│       ├── reg_file.sv
│       ├── uart.sv
│       ├── uart_rx.sv
│       ├── uart_tx.sv
│       └── top_uart.sv
└── uart-systemverilog-mod/
    ├── const/zybi-master.xdc
    └── src/
        ├── baud_gen.sv
        ├── uart_tx.sv
        ├── uart_rx.sv
        ├── uart.sv
        ├── top_uart.sv
        ├── reg_file.vhd
        ├── fifo_ex_stat.vhd
        └── fifo_ctrl_ex_stat.vhd
```

The XDC files are Zybo master constraints despite the `zybi-` filename typo.

---

## Architecture

All variants implement the same classic FSMD-style UART:

| Block | Purpose |
|---|---|
| `baud_gen` | Free-running counter that produces a one-clock `tick` every `dvsr + 1` clock cycles. |
| `uart_rx` | Receiver FSM (`idle → start → data → stop`), 16 ticks per bit, 2-flop synchronizer on `rx`, samples in the middle of each bit. |
| `uart_tx` | Transmitter FSM (`idle → start → data → stop`), LSB-first, 16 ticks per bit. |
| `fifo_ctrl` | Circular read/write pointer controller with `empty`/`full` flags. |
| `reg_file` | Small synchronous memory array used as FIFO storage. |
| `fifo` | `fifo_ctrl` + `reg_file`; suppresses writes when full. |
| `uart` | Top-level UART glue: baud generator, RX, TX, RX FIFO, TX FIFO. |
| `top_uart` | Board wrapper: drives LEDs from the low nibble of the most recently received byte. |

### Baud rate

The generator counts from `0` to `dvsr`, so the tick period is `dvsr + 1` clocks:

```text
baud = f_clk / (16 × (dvsr + 1))
```

With the Zybo 125 MHz clock and `dvsr = 68` as currently configured:

```text
125 MHz / (16 × 69) ≈ 113.2 kbaud
```

For approximately 115200 baud from 125 MHz, use `dvsr ≈ 67`. An exact 115200 baud would require a fractional/divider enhancement.

### Parameters

| Parameter | Default / used value | Meaning |
|---|---:|---|
| `DBIT` | 8 | Data bits. The core ports are currently hardcoded to 8 bits despite this generic. |
| `SB_TICK` | 16 | Ticks per bit / stop-bit length. |
| `FIFO_W` | 4 in core, but `0` in `top_uart` | FIFO address width; depth is `2**FIFO_W`. Use at least `2`. |
| `dvsr` | 68 | Baud-rate divisor. |
| `N` | 11 | Width of the divisor counter. |

### Board connections

From `const/zybi-master.xdc` (125 MHz, 8 ns clock):

| Signal | Pin | Notes |
|---|---|---|
| `clk` | K17 | `sysclk`, 125 MHz |
| `rst` | K18 | `btn[0]` |
| `led[3:0]` | M14, M15, G14, D18 | Discrete LEDs |
| `tx` | W16 | JE connector pin |
| `rx` | J15 | JE connector pin |

---

## Simulation

### VHDL variant

The VHDL testbenches use `std.env.stop`, so compile with VHDL-2008 or later.

Example with GHDL:

```powershell
ghdl -a --std=08 src\*.vhd
ghdl -a --std=08 sim\uart_tx_tb.vhd
ghdl -e --std=08 uart_tx_tb
ghdl -r --std=08 uart_tx_tb --stop-time=10us
```

Repeat with each `*_tb.vhd` file. `top_uart_tb.vhd` runs for 1 ms.

Example with Vivado XSim (run from the `uart-vhdl` directory after creating a project):

```tcl
add_files src/*.vhd
add_files -fileset sim_1 sim/*_tb.vhd

# Select one testbench at a time
set_property top uart_tx_tb [get_filesets sim_1]
launch_simulation
```

### SystemVerilog variants

There are no SystemVerilog testbenches in this tree. The VHDL testbenches are the best reference for expected timing and stimulus.

---

## Known issues

These are source-level findings, not simulation results.

1. **Top-level TX never starts.**  
   `wr_uart` and `rd_uart` are tied high at the same time in both `top_uart.vhd` and `top_uart.sv`. In `fifo_ctrl`, the simultaneous read/write case advances pointers but does not update `full`/`empty`, so `tx_empty` stays `1` and `tx_start = ~tx_empty` remains `0`.

2. **`FIFO_W = 0` in the top levels.**  
   This creates a degenerate one-entry FIFO (and a questionable zero-width address vector in SystemVerilog). Use `FIFO_W >= 2`.

3. **`w_data` is not driven in `uart-systemverilog/src/top_uart.sv`.**  
   The VHDL top drives a constant `x"56"`, so the LEDs show `0x6`.

4. **Baud divisor is off by one for 115200.**  
   See [Baud rate](#baud-rate).

5. **`uart-systemverilog-mod/` is unfinished.**  
   Non-exhaustive list:
   - `top_uart.sv`: missing comma after `.tx(tx)`, trailing comma in the `ld` concatenation.
   - `baud_gen.sv`: `tick` declared as `input` instead of `output`.
   - `uart.sv`: connects `.s_tick(s_tick)` although the signal is `tick`; connects RX FIFO `.wr(rx_data_out)` instead of `rx_done_tick`; declares `r_data` as input while driving it.
   - `uart_tx.sv`: invalid `always_comb @(...)`; drives `tx` directly and also via `assign tx = tx_reg`; output named `rx_done_tick`.
   - `uart_rx.sv`: malformed `done`/`dout` ports; increments `s_reg` when `s_tick` is low.

6. **RX has no stop-bit validation.**  
   It always asserts `rx_done_tick` after the stop tick.

7. **Testbenches are waveform-only.**  
   They do not contain assertions or scoreboards.

---

## Suggested cleanup work

- Tie `wr_uart`/`rd_uart` to real control pulses instead of constants.
- Set `FIFO_W` to at least `2` in the top levels.
- Correct `dvsr` for the target baud rate.
- Decide whether to keep only the VHDL version, only the SystemVerilog version, or finish the mixed-language status-extension variant.
- Add assertion-based RX/TX loopback and FIFO tests.
