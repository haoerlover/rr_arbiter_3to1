# rr_arbiter_3to1

Three-input round-robin arbiter with valid/ready handshakes.

## Contents

- `rtl/rr_arbiter_3to1.sv`: synthesizable SystemVerilog RTL.
- `tb/tb_rr_arbiter_3to1.sv`: self-checking testbench.
- `sim/Makefile`: VCS/Verdi compile, simulation, and waveform targets.
- `sim/flist`: source file list.
- `doc/`: LRS, design notes, stage records, self-check report, and waveform analysis.

## Basic Usage

```sh
cd sim
make compile
make sim
make sim_wave
```

`make sim_wave` generates `dump.fsdb` locally. Waveform and simulation build products are intentionally ignored by git.

## Interface Summary

The arbiter accepts three independent input channels:

- `in_valid_i[2:0]`
- `in_ready_o[2:0]`
- `in_data_i[2:0][DATA_WIDTH-1:0]`

It emits one output channel:

- `out_valid_o`
- `out_ready_i`
- `out_data_o`
- `grant_o[2:0]`

The output is registered. The round-robin pointer advances only when an output beat is consumed by `out_valid_o && out_ready_i`.
