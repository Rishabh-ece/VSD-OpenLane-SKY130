# 01 — Environment setup and OpenLANE preparation

[← Back to project README](../README.md)

## Objective and theory

A reproducible flow begins with a compatible container, PDK, and design configuration. OpenLANE orchestrates separate EDA tools and stores the outputs of each stage under a run directory.

## Commands / checkpoints

The following are representative commands; some screenshots come from different experiments, not one uninterrupted run.

```tcl
docker images --no-trunc
# From OpenLANE working directory, after mounting PDK and project:
./flow.tcl -interactive
prep -design picorv32a
```

## Screenshot-by-screenshot explanation

### Step 1 — Docker and OpenLANE startup

The local Docker image was checked and the OpenLANE v0.21 container was launched with the working directory and PDK mounted. The interactive Tcl prompt confirms the flow is running inside the container.

![Docker and OpenLANE startup](../images/1(1).png)

### Step 2 — PicoRV32A design configuration

The `config.tcl` file points OpenLANE to the PicoRV32A RTL and SDC, and defines the clock port and initial clock period. Configuration values control which design is synthesized and constrained.

![PicoRV32A design configuration](../images/2(1).png)

### Step 3 — SKY130-specific settings

The SKY130 HD configuration adjusts synthesis fanout, placement utilization, and placement density. These parameters affect area, congestion, and timing rather than changing the RTL functionality.

![SKY130-specific settings](../images/3(1).png)

### Step 4 — Prepare the design in OpenLANE

`prep -design picorv32a` loads the design, selects the SKY130 PDK, and prepares the run directories, technology libraries, and merged LEF views.

![Prepare the design in OpenLANE](../images/4(1).png)

### Step 5 — Inspect generated design files

The design directory contains the generic and PDK-specific Tcl configuration files, RTL sources, and generated run folders. This layout separates reusable inputs from generated implementation data.

![Inspect generated design files](../images/5(1).png)

### Step 6 — Merged LEF and macro definitions

The merged LEF file supplies physical size, layer, pin, and obstruction information used by placement and routing tools. Viewing a flip-flop macro verifies that the technology cells were included.

![Merged LEF and macro definitions](../images/6_macro_merged_lef(1).png)

### Step 7 — OpenLANE run directory

Each run stores `results`, `reports`, `logs`, and temporary artifacts separately. This makes it possible to inspect a particular experiment without mixing its outputs with a different run.

![OpenLANE run directory](../images/7_runs_dir(1).png)

### Step 8 — Resolved run configuration

The generated run `config.tcl` records the effective parameters, paths, and technology settings for that experiment. It is useful for reproducibility and debugging mismatched clock or library settings.

![Resolved run configuration](../images/8_runs_config_tcl(1).png)

## Key takeaway

The PDK and all paths must resolve correctly before running synthesis. Keep the generated run configuration for provenance.
