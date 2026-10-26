# VSD SKY130 — RTL-to-Physical Design with a Custom CMOS Inverter

> **VSD (VLSI System Design) practical project:** implementation of the `picorv32a` RISC-V processor using the SKY130 PDK and legacy OpenLANE, with a custom CMOS inverter designed in Magic and integrated into synthesis and placement.

This README is a **single, complete walkthrough** of the work. It explains the theory, the sequence of commands, what the screenshots show, the measured results and the issues encountered. It is deliberately designed to be read **top to bottom** on GitHub without following external documentation links. All **62 lab screenshots** are stored directly in `images/` (no subfolders).

## Project at a glance

| Item | Details |
|---|---|
| Design under implementation | `picorv32a` (PicoRV32 RISC-V core) |
| Process design kit | SkyWater SKY130 |
| Standard-cell library | `sky130_fd_sc_hd` (high-density cells) |
| Main flow | OpenLANE v0.21, legacy interactive Tcl workflow |
| Tools observed | Ubuntu/VirtualBox, Docker, Yosys, OpenSTA, OpenROAD, Magic, ngspice |
| Custom inverter | `sky130_vsdinv` |
| Evidence | 62 chronological screenshots in flat `images/` directory |
| Completion | Synthesis, floorplan, placement, CTS, PDN and routing reached success messages in documented runs; **timing violations remain** |

## Repository layout

```text
vsd-sky130-custom-cell-physical-design/
├── README.md
└── images/
    ├── 1(1).png
    ├── 2(1).png
    ├── ...
    └── 62_routing_success.png
```

> **Scope:** This repository documents the practical flow primarily through screenshots. It does not claim to include the original VM's complete RTL/PDK/run outputs, sign-off reports or a fabricated complete GDSII deliverable.

## End-to-end process

```text
PicoRV32A RTL + SDC + SKY130 libraries
                 |
                 v
       Yosys logic synthesis
       (gate-level Verilog)
                 |
                 v
  Floorplan (die/core, rows, pins, PDN)
                 |
                 v
      Standard-cell placement
                 |
                 v
    Timing analysis & optimization
                 |
                 v
        Clock tree synthesis
                 |
                 v
     Power grid and signal routing
                 |
                 v
    Timing / physical verification

Parallel custom-cell work:
CMOS inverter layout (Magic) -> extraction / SPICE
-> LEF + Liberty preparation -> library integration
-> repeat synthesis / floorplan / placement
-> verify `sky130_vsdinv` in placed layout
```

**Key terms:** RTL = register-transfer-level hardware description; PDK = process design kit; LEF = technology/library physical abstract; Liberty = timing and logic cell model; SDC = timing constraints; DEF = physical design coordinates; STA = static timing analysis; CTS = clock tree synthesis; PDN = power distribution network; WNS/TNS = worst/total negative timing slack.

---


## 1. Environment Setup, Docker and SKY130

### Why these tools are needed

A physical-design flow begins with **RTL** (a description of hardware behavior) and ends with a geometric implementation. **OpenLANE** coordinates several tools rather than doing every step itself. **Yosys** performs logic synthesis; **OpenROAD** handles floorplanning, placement, clock trees, routing and timing-related optimization; **OpenSTA** checks timing; **Magic** helps inspect and create layout; **ngspice** can simulate extracted transistor circuits. **SKY130** is the process design kit (PDK): it contains layer definitions, design rules and standard-cell models for the 130 nm technology.

Docker provides a consistent tool environment. The mounted **PDK** supplies the technology files while the mounted **OpenLANE directory** supplies the design RTL, configuration and generated runs. In screenshots, OpenLANE announces version `v0.21`; this is the legacy interactive flow, so commands may differ from modern OpenLane releases.

### Commands used / illustrated

```bash
# Run on Ubuntu host, from the OpenLANE checkout
docker images --no-trunc
# Find the local image ID if an expected tag cannot be found.
# Preserve your own PDK_ROOT environment variable.
docker run -it -v "$(pwd):/openLANE_flow" -v "$PDK_ROOT:$PDK_ROOT" \
  -e PDK_ROOT="$PDK_ROOT" -u "$(id -u):$(id -g)" <local-openlane-image-id>
```

```tcl
# Inside the container
cd /openLANE_flow
./flow.tcl -interactive
package require openlane 0.9
prep -design picorv32a
```

**What `prep` does:** it reads design-specific `config.tcl`, resolves the SKY130 libraries, creates a timestamped `runs/` directory and generates merged LEF/trimmed Liberty inputs. The directory normally contains `results/`, `reports/`, `logs/` and `tmp/`. Configurations in screenshots include *different clock periods at different stages*; use each run's saved `config.tcl` when comparing measurements.


### Lab screenshots and explanations

### Step 1 — Docker and OpenLANE startup

The local Docker image was checked and the OpenLANE v0.21 container was launched with the working directory and PDK mounted. The interactive Tcl prompt confirms the flow is running inside the container.

![Docker and OpenLANE startup](images/1(1).png)

### Step 2 — PicoRV32A design configuration

The `config.tcl` file points OpenLANE to the PicoRV32A RTL and SDC, and defines the clock port and initial clock period. Configuration values control which design is synthesized and constrained.

![PicoRV32A design configuration](images/2(1).png)

### Step 3 — SKY130-specific settings

The SKY130 HD configuration adjusts synthesis fanout, placement utilization, and placement density. These parameters affect area, congestion, and timing rather than changing the RTL functionality.

![SKY130-specific settings](images/3(1).png)

### Step 4 — Prepare the design in OpenLANE

`prep -design picorv32a` loads the design, selects the SKY130 PDK, and prepares the run directories, technology libraries, and merged LEF views.

![Prepare the design in OpenLANE](images/4(1).png)

### Step 5 — Inspect generated design files

The design directory contains the generic and PDK-specific Tcl configuration files, RTL sources, and generated run folders. This layout separates reusable inputs from generated implementation data.

![Inspect generated design files](images/5(1).png)

### Step 6 — Merged LEF and macro definitions

The merged LEF file supplies physical size, layer, pin, and obstruction information used by placement and routing tools. Viewing a flip-flop macro verifies that the technology cells were included.

![Merged LEF and macro definitions](images/6_macro_merged_lef(1).png)

### Step 7 — OpenLANE run directory

Each run stores `results`, `reports`, `logs`, and temporary artifacts separately. This makes it possible to inspect a particular experiment without mixing its outputs with a different run.

![OpenLANE run directory](images/7_runs_dir(1).png)

### Step 8 — Resolved run configuration

The generated run `config.tcl` records the effective parameters, paths, and technology settings for that experiment. It is useful for reproducibility and debugging mismatched clock or library settings.

![Resolved run configuration](images/8_runs_config_tcl(1).png)

---


## 2. RTL Synthesis and Baseline Timing

### Theory: behavioral RTL to real library gates

RTL statements such as `always @(posedge clk)` describe registers; combinational expressions describe logic. Synthesis **elaborates**, simplifies and **technology maps** this logic to gates and flip-flops present in the `sky130_fd_sc_hd` Liberty library. The output is a **gate-level Verilog netlist**, where instances have names such as `sky130_fd_sc_hd__dfxtp_2`, `sky130_fd_sc_hd__mux2_1` and `sky130_fd_sc_hd__or2_2`.

A **flip-flop** stores one bit between clock edges; combinational gates compute new values. The area report counts the mapped instances, not transistors. Timing reports estimate how long data takes to travel from a launch point through gates to a capture point. A successful synthesis message means a mapped netlist was produced—it does not imply the clock target was met.

### Commands and report locations

```tcl
# Run from the OpenLANE interactive % prompt after prep
run_synthesis
```

```bash
# Run from OpenLANE project directory on the host
ls designs/picorv32a/runs/<run_id>/results/synthesis/
ls designs/picorv32a/runs/<run_id>/reports/synthesis/
less designs/picorv32a/runs/<run_id>/reports/synthesis/1-yosys_4.stat.rpt
```

`<run_id>` is a placeholder for the **actual** timestamped run folder. Example report names are from the captured v0.21 flow. The baseline synthesis screenshot records **14,876 mapped cells**, including **1,613 `dfxtp_2` flip-flops**. Timing must be evaluated using the actual SDC and Liberty corners.


### Lab screenshots and explanations

### Step 1 — Successful synthesis

Yosys maps RTL operations and registers into SKY130 standard cells. The synthesis-success message confirms netlist creation; it does not by itself prove that timing constraints are met.

![Successful synthesis](images/9_syn_succes.png)

### Step 2 — Sequential logic statistics

The synthesis statistics enumerate mapped cell types and sequential elements. Flip-flops store processor state; combinational gates implement its datapath and control logic.

![Sequential logic statistics](images/10_flop_ratio.png)

### Step 3 — Generated synthesized netlist

The `results/synthesis` folder contains a synthesized Verilog netlist and a merged LEF reference. The netlist is the structural input for floorplanning and later physical-design stages.

![Generated synthesized netlist](images/11_results_synthesis.png)

### Step 4 — Inspect gate-level Verilog

The Yosys-generated Verilog lists intermediate wires and standard-cell instances rather than high-level RTL behavior. This illustrates technology mapping from source code to actual library cells.

![Inspect gate-level Verilog](images/12_extra_.vfile.png)

### Step 5 — Locate synthesis reports

Synthesis produces cell statistics and OpenSTA reports under `reports/synthesis`. Keeping the report filename and run directory allows timing results to be traced to a specific configuration.

![Locate synthesis reports](images/13_show_report_terminal.png)

### Step 6 — Cell count and mapping distribution

The baseline report shows **14,876 mapped cells**, including **1,613 `sky130_fd_sc_hd__dfxtp_2`** flip-flops. Cell distribution helps identify area usage and the ratio of sequential versus combinational logic.

![Cell count and mapping distribution](images/14_syn_report.png)

### Step 7 — Critical timing path

A max-delay timing report traces a register-to-register path through combinational cells. Arrival time versus required time determines setup slack; a negative value is a timing violation.

![Critical timing path](images/15_syn_timing_report.png)

### Step 8 — Synthesis parameter reference

The legacy OpenLANE configuration reference explains variables such as synthesis fanout, driving cell, capacitance load, and I/O timing assumptions. These settings should be chosen consistently with the intended design scenario.

![Synthesis parameter reference](images/16_confi_readme_syn.png)

---


## 3. Floorplanning, Pin Placement and PDN Preparation

### Theory: assigning space before cells and wires

The **die** is the overall chip boundary; the **core** is the interior region where rows of standard cells are placed. **Core utilization** is the fraction of core area expected to be occupied by placed logic; it trades chip size against whitespace needed for legalization, buffers and routing. High utilization can increase wire congestion and timing pressure. **Aspect ratio** sets the height-to-width shape of the core. **Placement rows** and **routing tracks** enforce legal manufacturing and library geometry.

The **LEF** (Library Exchange Format) file gives the place-and-route tool physical abstractions—cell sizes, pins, allowed layers and obstructions. The **DEF** (Design Exchange Format) file records design-specific geometry such as die area, cells, coordinates and pins. Floorplanning also assigns I/O pin positions. Pins affect routing distance; moving them can change congestion even though RTL stays identical.

**Power distribution network (PDN):** VDD and ground must be distributed via power rails and wider metal stripes. This is different from data-signal routing. Floorplan-stage power stripes are later checked for electrical connectivity.

### Common configuration and execution

```tcl
# Example study settings shown in the course screenshots
set ::env(FP_CORE_UTIL) 35
set ::env(PL_TARGET_DENSITY) [expr ($::env(FP_CORE_UTIL)+5) / 100.0]

# From OpenLANE interactive prompt
run_floorplan
```

```bash
# Inspect each run's output, not an unrelated earlier run
ls designs/picorv32a/runs/<run_id>/logs/floorplan/
ls designs/picorv32a/runs/<run_id>/results/floorplan/
```

The screenshots show the original configuration files, successful floorplan execution and later I/O experiments. `FP_IO_HMETAL` and `FP_IO_VMETAL` control preferred layers for horizontal and vertical edges; detailed behavior is specific to this older flow.


### Lab screenshots and explanations

### Step 1 — Floorplanning parameter reference

Floorplanning defines the die/core dimensions, target utilization, and placement of I/O pins. Core utilization is the share of core area intended for cells, leaving space for routing.

![Floorplanning parameter reference](images/17_confi_readme_fp.png)

### Step 2 — Floorplan defaults

The default floorplan Tcl file contains margins, aspect ratio, pin-layer choices, and PDN parameters. Reading defaults clarifies which dimensions are automatic and which are overridden.

![Floorplan defaults](images/18_confi_fp_tcl.png)

### Step 3 — Execute floorplanning

The floorplanning stage generates an initial die/core geometry and uses the synthesized cell areas to size the design. Successful completion produces a DEF database for subsequent stages.

![Execute floorplanning](images/19_run_fp.png)

### Step 4 — Inspect floorplan logs

Separate log files are generated for Verilog-to-DEF conversion, I/O pin placement, tap-cell insertion, and PDN creation. Reviewing individual logs helps localize warnings or errors.

![Inspect floorplan logs](images/20_logs_fp.png)

### Step 5 — Floorplan result files

Floorplan results are stored as intermediate DEF and related data files. DEF records the physical design, including die outline, placed cells, tracks, and pin locations.

![Floorplan result files](images/21_results_fp.png)

### Step 6 — Visualize floorplan

The physical viewer shows the floorplan geometry and boundary. At this stage the layout represents the area and structural setup; detailed signal routing is not yet complete.

![Visualize floorplan](images/22_fp_view.png)

### Step 7 — Review floorplan geometry

A second view helps inspect the distribution of floorplan objects and available placement space. It is useful for checking whether the core dimensions leave enough routing resources.

![Review floorplan geometry](images/23_fp_2.png)

### Step 8 — Adjust I/O pin placement

I/O placement assigns the design ports to suitable metal-layer boundaries. Pin position matters because it influences later signal routing and access from outside the core.

![Adjust I/O pin placement](images/27_set_io_2.png)

### Step 9 — Inspect updated I/O floorplan

The updated floorplan view shows the effects of the I/O configuration changes. This is a visual check of pin placement, not a substitute for a routing or DRC report.

![Inspect updated I/O floorplan](images/28_fp_io_2.png)

---


## 4. Placement and Visual Verification

### Theory: making the netlist physical

After synthesis, the netlist knows **which cells connect**, but not **where each cell sits**. Placement positions the instances in legal standard-cell rows, considering connectivity, density, wire length and overlap constraints. **Global placement** proposes approximate positions; **detailed placement / legalization** snaps cells to valid sites without illegal overlap. Placement is not routing: in an unrouted image, you can see many cell blocks even though metal signal connections are not yet completed.

A graphical view in Magic can be visually busy because thousands of cells and instance labels occupy a small area. Zooming lets us verify rows and find the custom inverter. The visible `sky130_vsdinv` instance in later screenshots is stronger evidence of physical integration than a synthesis-only success message.

### Run and inspect placement

```tcl
run_placement
```

Check the placement results and corresponding logs within the same `runs/<run_id>/` tree. The screenshots show both a baseline placement and a later placement after custom-cell integration. They should **not** be treated as the same experiment.


### Lab screenshots and explanations

### Step 1 — Run placement

Placement assigns locations to standard-cell instances while respecting row/site and legalization constraints. The goal is a routable arrangement with manageable wire length and congestion.

![Run placement](images/24_run_placement.png)

### Step 2 — Placed core overview

The resulting physical layout is densely populated with standard cells. Placement organizes the mapped netlist into rows; cell connections still require later routing.

![Placed core overview](images/25_placement.png)

### Step 3 — Inspect placement detail

Zooming into placement reveals individual cell locations and whitespace. Close inspection can expose overlaps, irregular rows, or cells of interest before routing.

![Inspect placement detail](images/26_placement_what.png)

### Step 4 — Floorplan succeeds with custom library

After the custom-cell integration changes, the floorplan stage successfully completes again. This verifies that the physical setup can progress with the updated technology inputs.

![Floorplan succeeds with custom library](images/44_run_fp_successfull.png)

### Step 5 — Placement succeeds with custom library

The global placement stage reports success in the custom-inverter run. A successful placement log verifies execution of this stage, not timing closure.

![Placement succeeds with custom library](images/45_run_placement_sccuess.png)

### Step 6 — Magic view of placed processor

Magic shows the full placed implementation at chip/core scale. This overview is useful for observing overall density before zooming into an individual library instance.

![Magic view of placed processor](images/46_placement_cell.png)

### Step 7 — Find `sky130_vsdinv` in placement

Zooming in reveals a `sky130_vsdinv` instance next to SKY130 standard cells. This is direct layout evidence that the custom cell has been incorporated into the placed design.

![Find `sky130_vsdinv` in placement](images/47_vsd_cell.png)

### Step 8 — Inspect inverter instance geometry

Expanding the highlighted cell shows its actual layout layers in context. It helps check that the custom inverter is aligned with the standard-cell rows and visible in the physical database.

![Inspect inverter instance geometry](images/48_expand_cell.png)

---


## 5. CMOS Inverter Design, Extraction and OpenLANE Integration

### Theory: how a CMOS inverter works

A CMOS inverter implements **Y = NOT(A)** using two complementary MOSFETs. The **PMOS** connects the output to the positive supply when input `A` is low; the **NMOS** connects output to ground when `A` is high. Ideally they are never fully on together in steady state, which reduces static power. During transitions, capacitances charge/discharge and both devices may briefly conduct, contributing switching delay and dynamic power.

| Input `A` | PMOS | NMOS | Output `Y` |
|---|---|---|---|
| 0 | ON | OFF | 1 |
| 1 | OFF | ON | 0 |

**Layout** is the geometric construction from diffusion, wells, polysilicon gates, contacts and metal. After drawing the transistor-level layout, **extraction** produces electrical connectivity and parasitic information; a **SPICE netlist** can then be inspected or simulated. For use in digital physical design, we need compatible abstract views:

- **LEF:** physical width/height, pin positions, signal/power layers and obstructions.
- **Liberty (`.lib`):** cell function, timing arcs, input capacitance, drive and power data.
- **SPICE:** transistor-level representation useful for circuit evaluation.

A technology-mapped cell requires consistent **cell and pin names** between LEF, Liberty and netlist. The screenshots show work with the `sky130_vsdinv` inverter, adjusting grid/size, exporting LEF, staging files, rerunning synthesis and locating instances in the placed design. They do not by themselves establish complete DRC/LVS sign-off or final silicon characterization.

### Magic extraction checkpoints

```tcl
# Representative Magic commands, executed inside Magic on the selected cell
extract all
ext2spice
```

### OpenLANE library integration checkpoints

1. Put the custom inverter's LEF and Liberty files in a known source location.
2. Reference the LEF in the OpenLANE design/technology configuration as appropriate for the flow version.
3. Make the Liberty model available to synthesis and timing analysis.
4. Rerun synthesis, then floorplanning and placement; inspect both netlist and layout for `sky130_vsdinv`.
5. Compare area and slack **using matching constraints and tool settings**, otherwise improvements cannot be assigned solely to the inverter.

**Note:** Code-level modifications and file contents not fully shown in the screenshots are intentionally not fabricated here. Use the original local files for a fully reproducible flow.


### Lab screenshots and explanations

### Step 1 — Create custom cell in Magic

The custom inverter is drawn using SKY130 layers and device structures. A CMOS inverter uses a PMOS pull-up and NMOS pull-down connected to a common input and output.

![Create custom cell in Magic](images/29_celldesign_layout.png)

### Step 2 — Inspect NMOS layout

The NMOS device is the pull-down path of the inverter. When its gate is high, it conducts and drives the output toward ground.

![Inspect NMOS layout](images/30_celldesign_nmos.png)

### Step 3 — Extract parasitic-aware SPICE view

Magic extraction converts the physical geometry into an electrical connectivity description. The extracted SPICE representation is useful for transistor-level checking and later characterization.

![Extract parasitic-aware SPICE view](images/31_extracting_spice_cell_design.png)

### Step 4 — Review extracted SPICE file

The SPICE text describes devices and connections obtained from layout extraction. Reviewing it helps catch disconnected terminals or unexpected devices before reuse.

![Review extracted SPICE file](images/32_vim_spice_file.png)

### Step 5 — Measure cell bounding box

The Magic box dimensions define the cell footprint. Accurate width and height are necessary for LEF-based row placement and alignment with the SKY130 site grid.

![Measure cell bounding box](images/33_dim_box_celldsiegn.png)

### Step 6 — Inspect routing-track information

Technology track pitch and layer conventions determine legal signal routing locations. The custom-cell pins need to be accessible from compatible routing layers.

![Inspect routing-track information](images/34_tracks_info.png)

### Step 7 — Align the layout grid

Adjusting the Magic grid helps match manufacturing or placement granularity. Grid alignment reduces geometry and accessibility mismatches when exporting a cell abstract.

![Align the layout grid](images/35_grid_change.png)

### Step 8 — Create a LEF abstract

LEF represents the inverter as a physical macro with size, pins, and geometry needed by the place-and-route engine. LEF does not replace the transistor-level SPICE view or timing Liberty model.

![Create a LEF abstract](images/36_cell_design_lef.png)

### Step 9 — Stage cell libraries in design sources

Custom-cell Liberty and LEF files were copied into the PicoRV32A design source area. The library supplies timing/electrical properties; LEF supplies geometric information.

![Stage cell libraries in design sources](images/37_copy_libs_src.png)

### Step 10 — Update configuration for custom cell

OpenLANE configuration and library paths were edited so synthesis and physical-design stages can locate the custom cell. Both logical and physical views must agree on the cell name and pins.

![Update configuration for custom cell](images/38_copyin_config_tcl.png)

### Step 11 — Rerun synthesis with custom library

Synthesis was repeated after library integration. A changed mapped netlist or cell mix must be compared within a consistent set of synthesis constraints.

![Rerun synthesis with custom library](images/39_custom_cell_synth.png)

### Step 12 — Modify synthesis settings

Synthesis options were adjusted during experimentation. Such changes can trade off area, fanout, and delay, so results should not be attributed to the new inverter alone unless settings are controlled.

![Modify synthesis settings](images/40_change_setting_synth.png)

### Step 13 — Inspect revised synthesis area

The post-change synthesis report records the resulting cell composition and area. Comparing report totals can show the effect of mapping options, but the screenshot alone does not establish a controlled benchmark.

![Inspect revised synthesis area](images/41_synth_after_area1.png)

### Step 14 — Compare before-and-after values

The experiment presents area figures from different runs. The comparison is useful as a learning exercise, with the caution that library and configuration differences can affect interpretation.

![Compare before-and-after values](images/42_before_after_area.png)

### Step 15 — Verify custom macro in merged LEF

The merged LEF was inspected to confirm that `sky130_vsdinv` appears in the physical library. Without a valid LEF macro, the place-and-route tool cannot place the cell correctly.

![Verify custom macro in merged LEF](images/43_less_merged_lef_vsd.png)

---


## 6. Static Timing Analysis and Cell Optimization

### Theory: setup, hold, slack and corners

**Static Timing Analysis (STA)** checks all constrained timing paths without test vectors. A **setup check** verifies data reaches a destination register sufficiently **before** its capturing clock edge. A **hold check** verifies data does not change too soon **after** that edge. In simplified form, `setup slack = required time - arrival time`; **negative slack means violation**. The reported delay depends on logic depth, fanout, cell size, wire parasitics, clock model and PVT/library corner.

- **WNS (Worst Negative Slack):** most negative slack among violating paths (or zero if none are negative, depending on report conventions).
- **TNS (Total Negative Slack):** sum of negative slacks for the reported endpoint set; more negative usually means broader timing trouble.
- **Fanout:** number of loads driven by one net; increased load can slow transitions.
- **Cell upsizing:** replace a weak gate with a stronger variant to lower output resistance; it also increases input capacitance, area and sometimes power.
- **Setup vs hold:** setup often benefits from faster data paths, while hold may need *extra* data delay. Fixing one can worsen the other.

**SDC (Synopsys Design Constraints)** defines clock periods and I/O timing requirements. The screenshot series contains **12.00 ns** and **24.73 ns** timing experiments; do not compare WNS across them as if the constraints were identical. Previous work clarifies the intended user SDC filename is `my_base.sdc`, not the older `base.sdc` placeholder visible in one screenshot.

### Example STA sequence

```tcl
# Commands used in an OpenSTA/OpenROAD-like environment
read_liberty -max /path/to/slow_corner.lib
read_liberty -min /path/to/fast_corner.lib
read_verilog /path/to/picorv32a.synthesis.v
link_design picorv32a
read_sdc /path/to/my_base.sdc
report_checks -path_delay max -fields {slew trans net cap input_pin}
report_wns
report_tns
```

Replace `/path/to/...` with real files. For setup analysis, use appropriate max libraries; for hold, appropriate min libraries. The visible comparison includes approximately **−37.00 ns** slack in one 12 ns experiment and approximately **−0.11 ns** in a later 24.73 ns placement experiment; these represent **different runs**, not a direct before/after of only one change.

### Timing optimization demonstration

```tcl
# Illustrative cell replacements from the captured experiment
replace_cell _13057_ sky130_fd_sc_hd__or4_4
replace_cell _13086_ sky130_fd_sc_hd__or3_4
replace_cell _13091_ sky130_fd_sc_hd__or4_4
replace_cell _13621_ sky130_fd_sc_hd__or2_4
replace_cell _13622_ sky130_fd_sc_hd__or3_4
report_checks -path_delay max
report_wns
report_tns
```

These internal instance names are **run-specific**; they should not be blindly reused with a different synthesized netlist. More negative TNS/WNS in later outputs must be documented as a remaining limitation, not a successful timing-closure claim.


### Lab screenshots and explanations

### Step 1 — Write STA command script

The STA script loads Liberty libraries, links a Verilog netlist, reads SDC constraints, and requests timing checks. STA estimates path delay without running a functional simulation.

![Write STA command script](images/49_create_sta_tcl.png)

### Step 2 — Inspect Liberty input capacitance

The Liberty input `A` capacitance is used when choosing realistic driving and loading assumptions. Pin capacitance affects cell delay and fanout loading.

![Inspect Liberty input capacitance](images/50_capA_for_basesdc.png)

### Step 3 — Adjust SDC timing assumptions

The SDC shown uses clock, input/output delay, driving-cell, and output-load constraints. The captured experiments include a 12 ns clock setting; older references to `base.sdc` should not be confused with your later `my_base.sdc` file.

![Adjust SDC timing assumptions](images/51_changed_base_sdc.png)

### Step 4 — Prepare standalone STA environment

The configuration provides library, netlist, and timing-constraint paths for OpenSTA. Consistent max/min library corners are important when interpreting setup and hold results.

![Prepare standalone STA environment](images/52_sta_pre_conf.png)

### Step 5 — Inspect negative setup slack

The timing report for a 12 ns constraint shows approximately **−37.00 ns** worst slack in one experiment. Negative slack means a path arrives later than its timing requirement.

![Inspect negative setup slack](images/53_red_slack_fanout4.png)

### Step 6 — Resize selected path cells

OpenSTA/OpenROAD `replace_cell` commands substitute higher-drive gate variants on a critical path. Upsizing can reduce cell delay but increases area and load on the preceding stage.

![Resize selected path cells](images/54_replacing_cells.png)

### Step 7 — Inspect timing using CTS outputs

After CTS, the actual clock tree can be included in path calculations. Reports from a propagated clock reflect insertion delay and skew not visible in an ideal-clock model.

![Inspect timing using CTS outputs](images/59_report_after_cts.png)

### Step 8 — Analyze skew and hold/setup

Clock skew is the difference in launch and capture clock arrival times. The screenshot reports approximately **0.46 ns** skew in one clock-tree comparison; setup and hold checks respond differently to skew.

![Analyze skew and hold/setup](images/60_setup_hold_skew.png)

---


## 7. Clock Tree Synthesis (CTS)

### Theory: routing the clock differently from data

A clock must reach thousands of sequential elements at nearly coordinated times. **CTS** builds a buffered branching network from the clock source to flip-flop clock pins. A **clock buffer** restores slew and drives groups of sinks; **clock insertion delay** is the elapsed time from the source to a sink; **skew** is the difference between clock arrival times at different sinks.

Before CTS the timing engine may use an **ideal clock**. After CTS, **propagated clocks** include clock-tree cell and interconnect delays, so setup/hold timing changes. **Positive or negative skew** can help one type of timing check while hurting another. The screenshot reports approximately **0.46 ns** skew for one pair of sinks; this is not a blanket skew figure for every clock endpoint.

### CTS commands and output verification

```tcl
run_cts
```

```tcl
# In timing analysis after building a clock network
set_propagated_clock [all_clocks]
report_clock_skew -setup
report_clock_skew -hold
report_checks -path_delay min_max
```

The CTS debug screenshots inspect two levels of Tcl code: the OpenLANE stage wrapper (which prepares inputs and outputs) and OpenROAD's `or_cts.tcl` (which creates the actual clock tree). Generated DEF/netlist/report paths are verified interactively. An `invalid command name` message visible in a prompt came from accidentally executing a bare cell-name string, rather than indicating CTS failure.


### Lab screenshots and explanations

### Step 1 — CTS completes successfully

Clock Tree Synthesis inserts a buffered clock distribution network. The success message confirms the tool ran; subsequent timing reports are needed to assess resulting skew and violations.

![CTS completes successfully](images/55_cts_success.png)

### Step 2 — Inspect OpenLANE CTS command flow

The CTS wrapper selects clock-net settings, trims/uses clock libraries, invokes OpenROAD, and writes DEF/netlist artifacts. This script is part of the automation linking placement to clock synthesis.

![Inspect OpenLANE CTS command flow](images/56_cts_tcl.png)

### Step 3 — Inspect OpenROAD CTS script

The OpenROAD script reads LEF, Liberty, DEF, Verilog, and SDC inputs, estimates parasitics, and calls the clock-tree synthesis engine with configured buffers.

![Inspect OpenROAD CTS script](images/57_or_cts_tcl.png)

### Step 4 — Confirm CTS output paths and buffer settings

Interactive Tcl inspection shows the generated CTS DEF, netlist, reports, and buffer configuration. The `invalid command name` line is from trying to execute an expanded cell name at the prompt, not proof CTS failed.

![Confirm CTS output paths and buffer settings](images/58_or_cts_echo.png)

---


## 8. Power Distribution, Routing and Final Checks

### Theory: power grid and signal connectivity

The **power distribution network (PDN)** connects the standard-cell power rails to broader metal stripes/rings and designated supply nets. Without a valid PDN, placed cells may be geometrically legal but not powered. **Power-grid checks** verify network connectivity; they are not equivalent to final IR-drop or electromigration sign-off. Some source locations may be moved to the nearest legal stripe by the PDN tool.

**Routing** then connects the logical signal nets using available metal/via resources while respecting spacing and design rules. **Global routing** decides approximate tracks and congestion; **detailed routing** chooses actual geometric wire segments. A routed DEF demonstrates completion of that stage, but a tapeout-quality design additionally requires closing timing, DRC, LVS and often antenna, IR-drop and EM checks.

### Execution and results

```tcl
# Representative legacy OpenLANE interactive commands
run_floorplan
run_placement
run_cts
run_routing
```

In the PDN log, warnings include a voltage on `VGND` not explicitly set and source points being moved to nearby stripes. The log also states **“PDN generation was successful.”** The final routing screenshot states **“Routing completed”** (run `26-10_01-33`, elapsed `0h12m45s`) with reported **WNS −15.76 ns** and **TNS −339.89 ns**. **Therefore routing completed, but timing closure was not demonstrated.**

#### Debugging a legacy PDN step

The course experimentation also modified `scripts/openroad/or_pdn.tcl` and inserted an informational message to confirm a workaround executed. Such a change is **version- and configuration-specific** and should be applied only when its source and behavior are reviewed. A successful PDN log is stronger evidence than the existence of a modified Tcl line.


### Lab screenshots and explanations

### Step 1 — Successful PDN generation with warnings

The power-grid tool reports successful PDN generation and connectivity, while also warning about moved via/stripe positions and power assumptions. These warnings should be reviewed rather than interpreted as sign-off.

![Successful PDN generation with warnings](images/61_pdn_gen.png)

### Step 2 — Routing completed, timing not closed

The final screenshot reports routing completed, with **WNS −15.76 ns** and **TNS −339.89 ns** for that run. Routing completion confirms progress through the flow but does not mean setup timing, DRC, or LVS closure.

![Routing completed, timing not closed](images/62_routing_success.png)

---


## Results and interpretation

| Checkpoint | Evidence / result | Interpretation |
|---|---|---|
| OpenLANE initial preparation | SKY130/HD libraries and merged LEF prepared | Environment resolves technology inputs |
| Baseline synthesis | **14,876** mapped cells; **1,613** `sky130_fd_sc_hd__dfxtp_2` flip-flops | Processor was technology mapped |
| Baseline floorplan / placement | Completed in recorded run | Design geometry and standard-cell positions generated |
| Custom inverter | `sky130_vsdinv` visible in Magic placed layout | Physical integration demonstrated |
| Timing experiments | Setup violations including ~**−37.00 ns** in a 12 ns run | Optimization is still required |
| Later placement experiment | WNS around **−0.11 ns** at 24.73 ns period (separate run) | This path nearly met the reported constraint; not total closure |
| CTS | Clock tree synthesis successful | Buffered clock network constructed |
| Clock skew example | Approximately **0.46 ns** between a reported pair | Clock arrival mismatch measured |
| PDN | Generation successful with warnings | Power network built; warnings should be reviewed |
| Routing | Completed in `26-10_01-33` in **0h12m45s** | Routing stage finished |
| Final route timing shown | **WNS −15.76 ns, TNS −339.89 ns** | **Timing not closed**; no final sign-off claim |

### Issues encountered and lessons learned

1. **Docker image-tag confusion:** the image `efabless/openlane:0.21` was not available under that exact tag, even though an existing local image ID could launch OpenLANE. Checking `docker images --no-trunc` helped identify the image.
2. **Different clock constraints:** screenshots of `config.tcl`, PDK configuration and run-time settings show various periods. Timing comparisons are meaningful only when the constraint, stage and corners are the same.
3. **Physical and timing views must match:** LEF alone describes geometry; Liberty supplies timing. A custom cell can appear in a layout while still requiring characterization and independent physical verification.
4. **Sizing trades area and timing:** swapping drive strengths or changing fanout can improve particular critical paths but may increase load, area, power, or introduce hold issues.
5. **PDN warnings need review:** a generated grid may still produce voltage/source-placement warnings. A success message means the stage completed, not that all supply-analysis requirements have been signed off.
6. **Routing completion is not tapeout:** negative WNS/TNS remain, and final DRC/LVS/IR-drop/EM signoff is not documented in these screenshots.

## What I learned

This lab connected the **logical view** of a CPU (RTL, mapped gates and timing arcs) with its **physical view** (rows, cell geometry, pins, metals and clocks). It also showed how introducing one custom standard cell requires several consistent representations—not just drawing its layout—and why timing analysis must be repeated after every major physical-design stage. The sequence of experiments is useful precisely because it includes both successful flow milestones and unresolved timing/debugging issues.

## Reproduction notes

- The images come from **multiple runs**, not one perfectly continuous run. Run names shown in screenshots include baseline timestamp folders, `custom_inv_test`, and later routing runs.
- Commands in this README are drawn from the documented workflow. Any command containing `<run_id>` or `/path/to/...` is a **template** to update, not a verbatim executable command.
- Legacy OpenLANE v0.21 and the matching SKY130 libraries are important for recreating the pictured output. Modern versions can have different script names, stages and parameters.
- The complete raw PDK, original source tree, extracted cell characterization inputs and final signoff artifacts are **not included** in the screenshot-only repository.

---

*VSD SKY130 physical-design course lab record — based on 62 screenshots and documented experiments.*
