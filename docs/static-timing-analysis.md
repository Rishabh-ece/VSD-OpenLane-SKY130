# 06 — Static timing analysis and optimization

[← Back to project README](../README.md)

## Objective and theory

**Static Timing Analysis (STA)** evaluates path delays against clock and I/O constraints without applying test vectors. Setup paths must arrive before their required time; negative slack indicates a violation. Cell sizing and fanout changes are possible optimization levers.

## Commands / checkpoints

The following are representative commands; some screenshots come from different experiments, not one uninterrupted run.

```tcl
report_checks -path_delay max
report_wns
report_tns
# Cell replacement was experimented with in OpenROAD
```

## Screenshot-by-screenshot explanation

### Step 1 — Write STA command script

The STA script loads Liberty libraries, links a Verilog netlist, reads SDC constraints, and requests timing checks. STA estimates path delay without running a functional simulation.

![Write STA command script](../images/49_create_sta_tcl.png)

### Step 2 — Inspect Liberty input capacitance

The Liberty input `A` capacitance is used when choosing realistic driving and loading assumptions. Pin capacitance affects cell delay and fanout loading.

![Inspect Liberty input capacitance](../images/50_capA_for_basesdc.png)

### Step 3 — Adjust SDC timing assumptions

The SDC shown uses clock, input/output delay, driving-cell, and output-load constraints. The captured experiments include a 12 ns clock setting; older references to `base.sdc` should not be confused with your later `my_base.sdc` file.

![Adjust SDC timing assumptions](../images/51_changed_base_sdc.png)

### Step 4 — Prepare standalone STA environment

The configuration provides library, netlist, and timing-constraint paths for OpenSTA. Consistent max/min library corners are important when interpreting setup and hold results.

![Prepare standalone STA environment](../images/52_sta_pre_conf.png)

### Step 5 — Inspect negative setup slack

The timing report for a 12 ns constraint shows approximately **−37.00 ns** worst slack in one experiment. Negative slack means a path arrives later than its timing requirement.

![Inspect negative setup slack](../images/53_red_slack_fanout4.png)

### Step 6 — Resize selected path cells

OpenSTA/OpenROAD `replace_cell` commands substitute higher-drive gate variants on a critical path. Upsizing can reduce cell delay but increases area and load on the preceding stage.

![Resize selected path cells](../images/54_replacing_cells.png)

### Step 7 — Inspect timing using CTS outputs

After CTS, the actual clock tree can be included in path calculations. Reports from a propagated clock reflect insertion delay and skew not visible in an ideal-clock model.

![Inspect timing using CTS outputs](../images/59_report_after_cts.png)

### Step 8 — Analyze skew and hold/setup

Clock skew is the difference in launch and capture clock arrival times. The screenshot reports approximately **0.46 ns** skew in one clock-tree comparison; setup and hold checks respond differently to skew.

![Analyze skew and hold/setup](../images/60_setup_hold_skew.png)

## Key takeaway

Several experiments show negative slack; the documented sizing and fanout investigations should not be presented as final timing closure.
