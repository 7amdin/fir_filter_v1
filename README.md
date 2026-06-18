# Sequential FIR Filter (Single-MAC Datapath)

VHDL implementation of a digital FIR filter built around a single multiply-accumulate (MAC) unit, driven by a small control FSM. Instead of instantiating one multiplier per tap (a fully parallel FIR), this design reuses one multiplier and one adder for every tap, cycling through them sequentially — a classic resource-shared DSP datapath.

## Top-level entity

`filtro_VLSIDSP_teste`

| Port    | Dir | Width | Description                          |
|---------|-----|-------|---------------------------------------|
| `clk`   | in  | 1     | System clock                          |
| `limpa` | in  | 1     | Synchronous clear / restart           |
| `X`     | in  | 8     | New input sample                      |
| `T`     | out | 16    | Filtered output (accumulator result)  |
| `cg`    | out | 1     | "Load" status flag from the FSM       |

## Building blocks

- **`somador16bits`** – 16-bit combinational adder (`s <= a + b`), used as the accumulator's addition stage.
- **`mult8bits`** – 8×8 → 16-bit combinational multiplier (sample × coefficient).
- **`reg16b`** – 16-bit positive-edge-triggered register that holds the running accumulation between cycles.
- **`rom`** – Parameterizable read-only memory (8 words × 8 bits in this configuration) that stores the filter coefficients, addressed by the FSM's pointer.
- **`mef`** – Finite-state machine with 8 states (`n0`…`n7`) that sequences the operation: each cycle it advances the ROM address (`app`), selects which delayed sample to use (`C`), and pulses the shift-register load signal (`ld`) once per full pass.
- **`mux8para3`** – 8-to-1, 8-bit multiplexer that picks the correct delayed input sample (tap) based on the FSM's selector.
- **`regdesl8b`** – 8-bit loadable register; eight of these are chained together to form the tapped delay line.
- **`registradordeslocamento`** – 8-stage shift register built from `regdesl8b`, holding the last 8 input samples (`Sa`…`Sh`).

## How it works

1. On each clock cycle, the FSM advances through states `n0`→`n7`, incrementing the ROM address and the tap selector together.
2. The ROM outputs the coefficient for the current tap; the multiplexer outputs the corresponding delayed sample from the shift register.
3. The multiplier computes `sample × coefficient`, which is added to the running total held in the accumulator register.
4. Once every 8 cycles, the shift register loads a new input sample `X`, pushing the oldest sample out and shifting all others down — implementing the FIR's delay line.
5. The accumulator output `T` represents the filtered result.

## Notes

- The ROM coefficients in this file are simple powers of two (`1, 2, 4, ..., 128`), which appear to be test/debug values rather than a real filter's coefficient set — useful for verifying the datapath before loading actual filter coefficients.
- All sub-components are packaged via `my_components` / `my_components1`, following the typical structural VHDL pattern of declaring components in a package and instantiating them with `port map`.
