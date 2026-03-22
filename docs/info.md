<!---

This file is used to generate your project datasheet. Please fill in the information below and delete any unused
sections.

You can also include images in this folder and reference them in the markdown. Each image must be less than
512 kb in size, and the combined size of all images must be less than 1 MB.
-->

## How it works

This block is the digital controller of an 8-phase buck converter. It manages startup, soft-start, run-time phase sequencing, and configuration through a serial shift register.

The controller receives external control and status signals, including the comparator output (`cot`), the soft-start end request (`hs_req`), and several enable/configuration pins. Internally, it generates the phase-driving outputs for up to 8 phases and exposes debug/status signals.

### Main functions

#### Startup state machine
The control flow is organized as a 4-state FSM:

- **IDLE**: waiting for controller enable.
- **DEADTIME**: after enable, the controller holds the comparator high for a fixed 1.6 ms delay.
- **SOFTSTART**: the switching activity is gradually introduced with a DDS-based ramp.
- **RUN**: normal operation.

If soft-start bypass is enabled, the controller skips DEADTIME and SOFTSTART and enters RUN directly.

#### Soft-start
During soft-start, the high-side pulse generation is not driven directly by the comparator. Instead, it is gated by a DDS ramp generator:
- the DDS starts from a programmable initial frequency,
- its increment increases over time,
- this progressively increases pulse density until soft-start ends.

This avoids an abrupt startup of the converter.

#### Phase sequencing
The controller supports from 1 to 8 active phases, configured by `cfg_n_ph`.

A phase counter rotates across the enabled phases and generates a one-hot pulse (`hs_onehot`) so that switching events are distributed round-robin across the active phases. This balances the load among the available power stages.

#### CCM / non-CCM behavior
The output behavior depends on `FORCE_CCM`:
- when CCM is enabled in RUN mode, only the active phase is pulsed one-hot,
- otherwise, all enabled phases are statically asserted as in the original non-forced mode.

Unused phases can optionally be forced low through `cfg_unused_force_low`.

#### Configuration interface
The block includes a 38-bit serial configuration shift register loaded with:
- soft-start bypass,
- external reference selection,
- soft-start initial frequency,
- TON selection,
- number of active phases,
- reference trim/source,
- unused-phase handling,
- compensation options,
- force CCM,
- debug mux selections.

Configuration data is shifted with `SR_CLK` / `SR_DATA` and committed with `SR_COMMIT`.

#### Debug outputs
Four output pins can be routed through a debug mux to observe internal signals such as:
- comparator input,
- soft-start state,
- run state,
- phase pulse generation,
- DDS tick,
- FSM state bits,
- control enables.

This simplifies bring-up and validation.

## How to test

The block can be tested in simulation by checking startup sequencing, soft-start behavior, phase rotation, and configuration handling.

### Basic startup test
1. Reset the design and keep `en_ctrl_1p2v` low.
2. Release reset and assert `en_ctrl_1p2v`.
3. Verify the FSM transitions:
   - `IDLE -> DEADTIME -> SOFTSTART -> RUN`
4. Check that:
   - `force_comp_high` is high only during DEADTIME,
   - `ss_active` is high only during SOFTSTART,
   - `ss_done` is high only during RUN.

### Soft-start bypass test
1. Enable `ss_bypass_ext` or set `cfg_ss_bypass`.
2. Assert `en_ctrl_1p2v`.
3. Verify that the controller skips DEADTIME and SOFTSTART and enters RUN directly.

### DDS soft-start test
1. Run with soft-start enabled.
2. Observe `tick`, `hs_pulse`, and `ss_active`.
3. Verify that:
   - DDS is initialized on SOFTSTART entry,
   - pulse activity starts at the programmed initial rate,
   - the effective pulse rate increases over time,
   - soft-start ends when `hs_req` is deasserted.

### Phase count test
1. Program different values of `cfg_n_ph` from 0 to 7.
2. Verify that the number of active phases is respectively 1 to 8.
3. In RUN mode, check that phase selection rotates only among enabled phases.
4. If `cfg_n_ph` changes dynamically, verify that `ph_count` is safely reset when out of range.

### Output mode test
1. Test with `cfg_fccm = 0`.
   - Verify that `FORCE_CCM` stays low.
   - Verify output behavior matches the non-forced mode.
2. Test with `cfg_fccm = 1`.
   - Verify that `FORCE_CCM` becomes high only in RUN.
   - Verify one-hot phase pulsing in RUN.

### Unused phase handling test
1. Configure fewer than 8 phases.
2. Test with `cfg_unused_force_low = 0` and `cfg_unused_force_low = 1`.
3. Verify the `hs_oe` behavior on unused phases in both cases.

### Configuration register test
1. Shift known data through `SR_DATA` using `SR_CLK`.
2. Assert `SR_COMMIT`.
3. Verify that the internal configuration fields are updated correctly:
   - TON selection,
   - phase count,
   - reference settings,
   - CCM mode,
   - debug mux selection.

### Debug mux test
1. Program each `dbg_sel_x` field with a known source.
2. Verify that `uo_out[4]` to `uo_out[7]` reflect the selected internal signals.

### Reset / disable test
1. Deassert enable during each FSM state.
2. Verify that the controller returns to IDLE and internal counters/registers are reset to their expected startup values.
