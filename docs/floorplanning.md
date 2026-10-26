# 03 — Floorplanning and I/O pin assignment

[← Back to project README](../README.md)

## Objective and theory

A **floorplan** establishes the physical size of the core/die and gives the implementation a coordinate system. I/O pin assignment, placement rows, routing tracks, and the power network must all be consistent.

## Commands / checkpoints

The following are representative commands; some screenshots come from different experiments, not one uninterrupted run.

```tcl
run_floorplan
# Inspect floorplan products:
ls designs/picorv32a/runs/<run_id>/results/floorplan
```

## Screenshot-by-screenshot explanation

### Step 1 — Floorplanning parameter reference

Floorplanning defines the die/core dimensions, target utilization, and placement of I/O pins. Core utilization is the share of core area intended for cells, leaving space for routing.

![Floorplanning parameter reference](../images/17_confi_readme_fp.png)

### Step 2 — Floorplan defaults

The default floorplan Tcl file contains margins, aspect ratio, pin-layer choices, and PDN parameters. Reading defaults clarifies which dimensions are automatic and which are overridden.

![Floorplan defaults](../images/18_confi_fp_tcl.png)

### Step 3 — Execute floorplanning

The floorplanning stage generates an initial die/core geometry and uses the synthesized cell areas to size the design. Successful completion produces a DEF database for subsequent stages.

![Execute floorplanning](../images/19_run_fp.png)

### Step 4 — Inspect floorplan logs

Separate log files are generated for Verilog-to-DEF conversion, I/O pin placement, tap-cell insertion, and PDN creation. Reviewing individual logs helps localize warnings or errors.

![Inspect floorplan logs](../images/20_logs_fp.png)

### Step 5 — Floorplan result files

Floorplan results are stored as intermediate DEF and related data files. DEF records the physical design, including die outline, placed cells, tracks, and pin locations.

![Floorplan result files](../images/21_results_fp.png)

### Step 6 — Visualize floorplan

The physical viewer shows the floorplan geometry and boundary. At this stage the layout represents the area and structural setup; detailed signal routing is not yet complete.

![Visualize floorplan](../images/22_fp_view.png)

### Step 7 — Review floorplan geometry

A second view helps inspect the distribution of floorplan objects and available placement space. It is useful for checking whether the core dimensions leave enough routing resources.

![Review floorplan geometry](../images/23_fp_2.png)

### Step 8 — Adjust I/O pin placement

I/O placement assigns the design ports to suitable metal-layer boundaries. Pin position matters because it influences later signal routing and access from outside the core.

![Adjust I/O pin placement](../images/27_set_io_2.png)

### Step 9 — Inspect updated I/O floorplan

The updated floorplan view shows the effects of the I/O configuration changes. This is a visual check of pin placement, not a substitute for a routing or DRC report.

![Inspect updated I/O floorplan](../images/28_fp_io_2.png)

## Key takeaway

The floorplan generated the core infrastructure and a DEF representation for subsequent implementation.
