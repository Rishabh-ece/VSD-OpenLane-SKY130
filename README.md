# VSD SKY130 — Custom Cell Physical Design

**RTL-to-physical-design implementation of the PicoRV32A RISC-V core using OpenLANE and the SKY130 PDK, including the design and integration of a custom CMOS inverter.**

This repository documents my practical work in the **VSD (VLSI System Design)** physical design course. It covers synthesis, floorplanning, placement, custom-cell design, timing optimization, clock tree synthesis, power distribution, and routing.

> **Note:** The screenshots capture multiple OpenLANE runs and timing experiments. This is an educational implementation log, **not** a timing-closed or tapeout-signoff claim.

## Project information

| Item | Details |
|---|---|
| Design | `picorv32a` — RISC-V processor core |
| Process / cells | SkyWater SKY130 · `sky130_fd_sc_hd` |
| Flow | OpenLANE v0.21 (legacy interactive Tcl flow) |
| Tools | Docker, Yosys, OpenSTA, OpenROAD, Magic, ngspice |
| Custom cell | `sky130_vsdinv` inverter |
| Evidence | 62 original lab screenshots in a **flat** `images/` folder |

## Physical design flow

`Environment → RTL Synthesis → Floorplan → Placement → Custom Inverter Integration → STA / Optimization → CTS → PDN → Routing`

## Labs / Tasks


<details>
<summary><b>Task 1 — Environment setup and OpenLANE preparation</b></summary>

### Objective

Configured Docker and SKY130, launched OpenLANE interactively, and checked generated run paths and merged libraries.

### Concept

A reproducible flow begins with a compatible container, PDK, and design configuration. OpenLANE orchestrates separate EDA tools and stores the outputs of each stage under a run directory.

### Evidence

![Environment setup and OpenLANE preparation](images/8_runs_config_tcl(1).png)

**8 screenshots explained:** [1](docs/environment-setup.md#step-1--docker-and-openlane-startup), [2](docs/environment-setup.md#step-2--picorv32a-design-configuration), [3](docs/environment-setup.md#step-3--sky130-specific-settings), [4](docs/environment-setup.md#step-4--prepare-the-design-in-openlane), [5](docs/environment-setup.md#step-5--inspect-generated-design-files), [6](docs/environment-setup.md#step-6--merged-lef-and-macro-definitions), [7](docs/environment-setup.md#step-7--openlane-run-directory), [8](docs/environment-setup.md#step-8--resolved-run-configuration)

**[Read the full illustrated walkthrough →](docs/environment-setup.md)**

</details>

<details>
<summary><b>Task 2 — RTL synthesis and baseline timing</b></summary>

### Objective

Mapped PicoRV32A RTL to SKY130 standard cells with Yosys and inspected cell count, flip-flops, and critical paths.

### Concept

**RTL synthesis** translates behavioral Verilog into an interconnected gate-level netlist. Mapping uses SKY130 library cells and timing constraints to choose an implementation; STA then identifies potentially slow paths.

### Evidence

![RTL synthesis and baseline timing](images/16_confi_readme_syn.png)

**8 screenshots explained:** [9](docs/synthesis.md#step-1--successful-synthesis), [10](docs/synthesis.md#step-2--sequential-logic-statistics), [11](docs/synthesis.md#step-3--generated-synthesized-netlist), [12](docs/synthesis.md#step-4--inspect-gate-level-verilog), [13](docs/synthesis.md#step-5--locate-synthesis-reports), [14](docs/synthesis.md#step-6--cell-count-and-mapping-distribution), [15](docs/synthesis.md#step-7--critical-timing-path), [16](docs/synthesis.md#step-8--synthesis-parameter-reference)

**[Read the full illustrated walkthrough →](docs/synthesis.md)**

</details>

<details>
<summary><b>Task 3 — Floorplanning and I/O pin assignment</b></summary>

### Objective

Generated the die/core floorplan, inspected OpenLANE defaults, and reviewed placement of I/O pins and the PDN.

### Concept

A **floorplan** establishes the physical size of the core/die and gives the implementation a coordinate system. I/O pin assignment, placement rows, routing tracks, and the power network must all be consistent.

### Evidence

![Floorplanning and I/O pin assignment](images/28_fp_io_2.png)

**9 screenshots explained:** [17](docs/floorplanning.md#step-1--floorplanning-parameter-reference), [18](docs/floorplanning.md#step-2--floorplan-defaults), [19](docs/floorplanning.md#step-3--execute-floorplanning), [20](docs/floorplanning.md#step-4--inspect-floorplan-logs), [21](docs/floorplanning.md#step-5--floorplan-result-files), [22](docs/floorplanning.md#step-6--visualize-floorplan), [23](docs/floorplanning.md#step-7--review-floorplan-geometry), [27](docs/floorplanning.md#step-8--adjust-io-pin-placement), [28](docs/floorplanning.md#step-9--inspect-updated-io-floorplan)

**[Read the full illustrated walkthrough →](docs/floorplanning.md)**

</details>

<details>
<summary><b>Task 4 — Placement and visualization</b></summary>

### Objective

Ran global placement, inspected Magic layout views, and located the integrated custom inverter.

### Concept

**Placement** puts the mapped cells at legal coordinates within the core. At this stage, cells are positioned but the detailed signal wires are not yet routed.

### Evidence

![Placement and visualization](images/48_expand_cell.png)

**8 screenshots explained:** [24](docs/placement.md#step-1--run-placement), [25](docs/placement.md#step-2--placed-core-overview), [26](docs/placement.md#step-3--inspect-placement-detail), [44](docs/placement.md#step-4--floorplan-succeeds-with-custom-library), [45](docs/placement.md#step-5--placement-succeeds-with-custom-library), [46](docs/placement.md#step-6--magic-view-of-placed-processor), [47](docs/placement.md#step-7--find-sky130vsdinv-in-placement), [48](docs/placement.md#step-8--inspect-inverter-instance-geometry)

**[Read the full illustrated walkthrough →](docs/placement.md)**

</details>

<details>
<summary><b>Task 5 — Custom inverter design and library integration</b></summary>

### Objective

Designed an inverter in Magic, extracted SPICE, created/inspected LEF, and connected physical and timing library views.

### Concept

A **CMOS inverter** uses a PMOS pull-up and NMOS pull-down: low input produces high output, and high input produces low output. To integrate it into an automated flow, the custom cell needs a physical abstract (LEF) and a logical/timing description (Liberty), with naming and pin consistency.

### Evidence

![Custom inverter design and library integration](images/43_less_merged_lef_vsd.png)

**15 screenshots explained:** [29](docs/custom-inverter-integration.md#step-1--create-custom-cell-in-magic), [30](docs/custom-inverter-integration.md#step-2--inspect-nmos-layout), [31](docs/custom-inverter-integration.md#step-3--extract-parasitic-aware-spice-view), [32](docs/custom-inverter-integration.md#step-4--review-extracted-spice-file), [33](docs/custom-inverter-integration.md#step-5--measure-cell-bounding-box), [34](docs/custom-inverter-integration.md#step-6--inspect-routing-track-information), [35](docs/custom-inverter-integration.md#step-7--align-the-layout-grid), [36](docs/custom-inverter-integration.md#step-8--create-a-lef-abstract), [37](docs/custom-inverter-integration.md#step-9--stage-cell-libraries-in-design-sources), [38](docs/custom-inverter-integration.md#step-10--update-configuration-for-custom-cell), [39](docs/custom-inverter-integration.md#step-11--rerun-synthesis-with-custom-library), [40](docs/custom-inverter-integration.md#step-12--modify-synthesis-settings), [41](docs/custom-inverter-integration.md#step-13--inspect-revised-synthesis-area), [42](docs/custom-inverter-integration.md#step-14--compare-before-and-after-values), [43](docs/custom-inverter-integration.md#step-15--verify-custom-macro-in-merged-lef)

**[Read the full illustrated walkthrough →](docs/custom-inverter-integration.md)**

</details>

<details>
<summary><b>Task 6 — Static timing analysis and optimization</b></summary>

### Objective

Built timing constraints and STA scripts, investigated negative slack, and experimented with drive-strength replacement.

### Concept

**Static Timing Analysis (STA)** evaluates path delays against clock and I/O constraints without applying test vectors. Setup paths must arrive before their required time; negative slack indicates a violation. Cell sizing and fanout changes are possible optimization levers.

### Evidence

![Static timing analysis and optimization](images/60_setup_hold_skew.png)

**8 screenshots explained:** [49](docs/static-timing-analysis.md#step-1--write-sta-command-script), [50](docs/static-timing-analysis.md#step-2--inspect-liberty-input-capacitance), [51](docs/static-timing-analysis.md#step-3--adjust-sdc-timing-assumptions), [52](docs/static-timing-analysis.md#step-4--prepare-standalone-sta-environment), [53](docs/static-timing-analysis.md#step-5--inspect-negative-setup-slack), [54](docs/static-timing-analysis.md#step-6--resize-selected-path-cells), [59](docs/static-timing-analysis.md#step-7--inspect-timing-using-cts-outputs), [60](docs/static-timing-analysis.md#step-8--analyze-skew-and-holdsetup)

**[Read the full illustrated walkthrough →](docs/static-timing-analysis.md)**

</details>

<details>
<summary><b>Task 7 — Clock tree synthesis</b></summary>

### Objective

Ran CTS and examined scripts, clock-buffer configuration, setup/hold clock paths, and skew.

### Concept

**CTS** balances and buffers the clock network so that registers receive clocks with controlled insertion delay and skew. Once the clock is propagated, real clock-path effects modify timing analysis.

### Evidence

![Clock tree synthesis](images/58_or_cts_echo.png)

**4 screenshots explained:** [55](docs/clock-tree-synthesis.md#step-1--cts-completes-successfully), [56](docs/clock-tree-synthesis.md#step-2--inspect-openlane-cts-command-flow), [57](docs/clock-tree-synthesis.md#step-3--inspect-openroad-cts-script), [58](docs/clock-tree-synthesis.md#step-4--confirm-cts-output-paths-and-buffer-settings)

**[Read the full illustrated walkthrough →](docs/clock-tree-synthesis.md)**

</details>

<details>
<summary><b>Task 8 — Power distribution and routing</b></summary>

### Objective

Examined power-grid warning messages, successful PDN generation, and the routing-completion log.

### Concept

The **PDN** distributes power and ground through stripes, rails, and vias. **Routing** connects signal nets over legal metal tracks; routing completion is distinct from final timing and physical verification.

### Evidence

![Power distribution and routing](images/62_routing_success.png)

**2 screenshots explained:** [61](docs/pdn-and-routing.md#step-1--successful-pdn-generation-with-warnings), [62](docs/pdn-and-routing.md#step-2--routing-completed-timing-not-closed)

**[Read the full illustrated walkthrough →](docs/pdn-and-routing.md)**

</details>

## Key observations and results

| Stage / run | Evidence-based result |
|---|---|
| Baseline synthesis | **14,876 mapped cells**, including **1,613 `dfxtp_2`** flip-flops in the captured report |
| Floorplanning / placement | Both stages completed in documented runs |
| Custom inverter | `sky130_vsdinv` visible among placed standard cells in Magic |
| STA experiments | Negative setup slack observed, including **−37.00 ns** in a 12 ns clock experiment |
| Clock tree synthesis | CTS reported successful; clock path and skew reports inspected |
| PDN | Generation reported successful, with warnings requiring review |
| Routing | Reported complete; run ended with **WNS −15.76 ns** and **TNS −339.89 ns** |

*Metrics come from different captured runs and are not directly comparable without matching configurations.*

## Repository organization

```text
vsd-sky130-custom-cell-physical-design/
├── README.md
├── images/                         # 62 screenshots, directly in this folder
│   ├── 1(1).png
│   ├── 2(1).png
│   ├── ...
│   └── 62_routing_success.png
├── docs/
│   ├── environment-setup.md
│   ├── synthesis.md
│   ├── floorplanning.md
│   ├── placement.md
│   ├── custom-inverter-integration.md
│   ├── static-timing-analysis.md
│   ├── clock-tree-synthesis.md
│   └── pdn-and-routing.md
└── (optional: original Tcl/SDC, LEF/Liberty, reports, and results)
```

## Lessons learned

- A physical design flow is a series of interdependent stages: correct libraries, netlists, constraints, and design files must be carried forward.
- LEF and Liberty models provide different, complementary views of a custom standard cell.
- Timing numbers depend on clock constraints, cell mapping, fanout, sizing, and clock propagation.
- **Successful placement, CTS, PDN, or routing is not equivalent to timing closure or sign-off.**

## Limitations

The screenshots do not establish DRC/LVS sign-off, post-route timing closure, or silicon fabrication. To make the work fully reproducible, the original edited Tcl and SDC scripts, custom-cell LEF/Liberty/SPICE, generated reports, and exact PDK/tool revisions should also be committed when available.

## Acknowledgements

Completed as part of the VSD physical design learning exercises using the open-source SKY130 PDK and EDA ecosystem.

