# 05 — Custom inverter design and library integration

[← Back to project README](../README.md)

## Objective and theory

A **CMOS inverter** uses a PMOS pull-up and NMOS pull-down: low input produces high output, and high input produces low output. To integrate it into an automated flow, the custom cell needs a physical abstract (LEF) and a logical/timing description (Liberty), with naming and pin consistency.

## Commands / checkpoints

The following are representative commands; some screenshots come from different experiments, not one uninterrupted run.

```tcl
# In Magic, inspect the cell and extract a SPICE netlist
extract all
ext2spice
# Confirm the exported LEF macro and corresponding Liberty cell
```

## Screenshot-by-screenshot explanation

### Step 1 — Create custom cell in Magic

The custom inverter is drawn using SKY130 layers and device structures. A CMOS inverter uses a PMOS pull-up and NMOS pull-down connected to a common input and output.

![Create custom cell in Magic](../images/29_celldesign_layout.png)

### Step 2 — Inspect NMOS layout

The NMOS device is the pull-down path of the inverter. When its gate is high, it conducts and drives the output toward ground.

![Inspect NMOS layout](../images/30_celldesign_nmos.png)

### Step 3 — Extract parasitic-aware SPICE view

Magic extraction converts the physical geometry into an electrical connectivity description. The extracted SPICE representation is useful for transistor-level checking and later characterization.

![Extract parasitic-aware SPICE view](../images/31_extracting_spice_cell_design.png)

### Step 4 — Review extracted SPICE file

The SPICE text describes devices and connections obtained from layout extraction. Reviewing it helps catch disconnected terminals or unexpected devices before reuse.

![Review extracted SPICE file](../images/32_vim_spice_file.png)

### Step 5 — Measure cell bounding box

The Magic box dimensions define the cell footprint. Accurate width and height are necessary for LEF-based row placement and alignment with the SKY130 site grid.

![Measure cell bounding box](../images/33_dim_box_celldsiegn.png)

### Step 6 — Inspect routing-track information

Technology track pitch and layer conventions determine legal signal routing locations. The custom-cell pins need to be accessible from compatible routing layers.

![Inspect routing-track information](../images/34_tracks_info.png)

### Step 7 — Align the layout grid

Adjusting the Magic grid helps match manufacturing or placement granularity. Grid alignment reduces geometry and accessibility mismatches when exporting a cell abstract.

![Align the layout grid](../images/35_grid_change.png)

### Step 8 — Create a LEF abstract

LEF represents the inverter as a physical macro with size, pins, and geometry needed by the place-and-route engine. LEF does not replace the transistor-level SPICE view or timing Liberty model.

![Create a LEF abstract](../images/36_cell_design_lef.png)

### Step 9 — Stage cell libraries in design sources

Custom-cell Liberty and LEF files were copied into the PicoRV32A design source area. The library supplies timing/electrical properties; LEF supplies geometric information.

![Stage cell libraries in design sources](../images/37_copy_libs_src.png)

### Step 10 — Update configuration for custom cell

OpenLANE configuration and library paths were edited so synthesis and physical-design stages can locate the custom cell. Both logical and physical views must agree on the cell name and pins.

![Update configuration for custom cell](../images/38_copyin_config_tcl.png)

### Step 11 — Rerun synthesis with custom library

Synthesis was repeated after library integration. A changed mapped netlist or cell mix must be compared within a consistent set of synthesis constraints.

![Rerun synthesis with custom library](../images/39_custom_cell_synth.png)

### Step 12 — Modify synthesis settings

Synthesis options were adjusted during experimentation. Such changes can trade off area, fanout, and delay, so results should not be attributed to the new inverter alone unless settings are controlled.

![Modify synthesis settings](../images/40_change_setting_synth.png)

### Step 13 — Inspect revised synthesis area

The post-change synthesis report records the resulting cell composition and area. Comparing report totals can show the effect of mapping options, but the screenshot alone does not establish a controlled benchmark.

![Inspect revised synthesis area](../images/41_synth_after_area1.png)

### Step 14 — Compare before-and-after values

The experiment presents area figures from different runs. The comparison is useful as a learning exercise, with the caution that library and configuration differences can affect interpretation.

![Compare before-and-after values](../images/42_before_after_area.png)

### Step 15 — Verify custom macro in merged LEF

The merged LEF was inspected to confirm that `sky130_vsdinv` appears in the physical library. Without a valid LEF macro, the place-and-route tool cannot place the cell correctly.

![Verify custom macro in merged LEF](../images/43_less_merged_lef_vsd.png)

## Key takeaway

The custom inverter was designed, abstracted, and located in the placed implementation. Use original SPICE/LEF/Liberty files for reproducible integration.
