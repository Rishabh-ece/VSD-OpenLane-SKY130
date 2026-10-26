# 07 — Clock tree synthesis

[← Back to project README](../README.md)

## Objective and theory

**CTS** balances and buffers the clock network so that registers receive clocks with controlled insertion delay and skew. Once the clock is propagated, real clock-path effects modify timing analysis.

## Commands / checkpoints

The following are representative commands; some screenshots come from different experiments, not one uninterrupted run.

```tcl
run_cts
# Inspect CTS outputs in results/cts and reports/cts
```

## Screenshot-by-screenshot explanation

### Step 1 — CTS completes successfully

Clock Tree Synthesis inserts a buffered clock distribution network. The success message confirms the tool ran; subsequent timing reports are needed to assess resulting skew and violations.

![CTS completes successfully](../images/55_cts_success.png)

### Step 2 — Inspect OpenLANE CTS command flow

The CTS wrapper selects clock-net settings, trims/uses clock libraries, invokes OpenROAD, and writes DEF/netlist artifacts. This script is part of the automation linking placement to clock synthesis.

![Inspect OpenLANE CTS command flow](../images/56_cts_tcl.png)

### Step 3 — Inspect OpenROAD CTS script

The OpenROAD script reads LEF, Liberty, DEF, Verilog, and SDC inputs, estimates parasitics, and calls the clock-tree synthesis engine with configured buffers.

![Inspect OpenROAD CTS script](../images/57_or_cts_tcl.png)

### Step 4 — Confirm CTS output paths and buffer settings

Interactive Tcl inspection shows the generated CTS DEF, netlist, reports, and buffer configuration. The `invalid command name` line is from trying to execute an expanded cell name at the prompt, not proof CTS failed.

![Confirm CTS output paths and buffer settings](../images/58_or_cts_echo.png)

## Key takeaway

CTS ran successfully, and clock reports were inspected for insertion delay, skew, and timing impact.

