# SoC Physical Design with OpenLANE

Notes and reference material from a physical-design workshop covering the open-source RTL-to-GDSII flow using OpenLANE, OpenROAD, and the SkyWater 130nm (sky130) PDK.

---

## 1. The Core Equation

Everything in digital ASIC design boils down to three ingredients:
<img width="508" height="526" alt="image" src="https://github.com/user-attachments/assets/f23f3779-68a6-4d4d-af22-d897c67424a0" />


| Ingredient    | What it is                                        |
| ------------- | -------------------------------------------------- |
| **RTL IP's**  | Your Verilog/HDL design or reused IP blocks         |
| **EDA Tools** | Software that turns RTL into physical layout        |
| **PDK Data**  | The "interface" between the fab and the designer    |

Historically all three were **closed-source**. The open-source movement (Google + SkyWater, OpenROAD, OpenLANE) opened up all three for the first time, enabling a fully open RTL-to-GDSII flow.

---

## 2. What is a PDK? (Process Design Kit)

A PDK is a collection of files that model a fabrication process for EDA tools — it's what lets you design a chip *without* being the fab.

| Contains                        | Purpose                                       |
| -------------------------------- | ---------------------------------------------- |
| Process Design Rules             | DRC, LVS, PEX                                  |
| Device Models                    | Transistor-level electrical behavior           |
| Digital Standard Cell Libraries  | Pre-built logic gates (AND, NAND, XOR, etc.)   |
| I/O Libraries                    | Pad/interface cells                            |

**Milestone:** June 30, 2020 — Google + SkyWater released **SKY130**, the first fully open-source production PDK (Apache 2.0, 130nm node). It's the PDK OpenLANE is tuned for: [github.com/google/skywater-pdk](https://github.com/google/skywater-pdk).

> 💡 **Is 130nm too old to matter?** No — it's still ~13% of foundry sales by volume, and designs like a single-cycle RV32I have hit 327 MHz post-layout (>1GHz pipelined) on SKY130.

### The sky130 PDK ecosystem (three layers)

- **skywater-pdk** – raw PDK files as released by the foundry
- **open_pdks** – scripts that convert the raw foundry files into a usable format
- **sky130A** – the final, ready-to-use PDK that EDA tools actually run with

Inside sky130A:

- **libs.tech** – tool-specific technology files (Magic tech file, KLayout layer/DRC rules, ngspice models) that tell each EDA tool how to interpret the process technology
- **libs.ref** – files specific to the standard cell library used in this workshop, `sky130_fd_sc_hd` (130nm, high-density variant for tight packing)

**File types inside `libs.ref`:**

| File type | Purpose                                                                              |
| --------- | ------------------------------------------------------------------------------------- |
| `verilog` | Behavioral models of each cell (functional simulation, not physical)                  |
| `spice`   | Transistor-level SPICE netlists for circuit-accurate simulation                       |
| `techlef` | Technology LEF — routing layers, vias, and design rules for the process               |
| `maglef`  | Magic-tool layout view of cells in LEF-compatible abstract form                       |
| `mag`     | Native Magic layout files (full geometric layout)                                     |
| `gds`     | GDSII files — actual physical layout used for fabrication                             |
| `doc`     | Datasheets / documentation for the cells                                              |
| `cdl`     | Circuit Description Language netlist, used for LVS checks                             |
| `lib`     | Liberty files — timing, power, and functional data used by synthesis/STA tools        |
| `lef`     | Abstract physical view of each cell (pins, blockages, size) for placement & routing    |

---

## 3. The RTL → GDSII Flow

This is the backbone of physical design — memorize this pipeline order before anything else:

```
Synthesis → Floorplan/Power Plan → Placement → CTS → Routing → Sign-off
```

| Stage                             | What happens                                                                                          | Key detail                                                                                              | Tool               |
| ---------------------------------- | -------------------------------------------------------------------------------------------------------| ----------------------------------------------------------------------------------------------------------| ------------------- |
| **1. Synthesis**                  | RTL → gate-level netlist using standard cells                                                          | Each std cell has multiple views: electrical, HDL, SPICE, layout (abstract + detailed)                    | Yosys + ABC        |
| **2. Floor & Power Planning**     | Partition the die among blocks, place I/O pads, define macro dimensions/pins/rows, build power grid    | Two sub-steps: chip floorplanning (macro-level) + power planning (VDD/VSS rings, straps, pads)             | OpenROAD           |
| **3. Placement**                  | Place gate-level cells onto floorplan rows, aligned to sites                                           | Done in 2 passes: Global → Detailed                                                                       | OpenROAD           |
| **4. Clock Tree Synthesis (CTS)** | Build a clock distribution network to every flip-flop                                                  | Goal: minimum skew (zero is unrealistic); usually an H-tree or X-tree shape                                | OpenROAD           |
| **5. Routing**                    | Wire up the netlist using metal layers                                                                 | Huge routing grid → divide & conquer: global routing (guides) → detailed routing (actual wires)            | OpenROAD           |
| **6. Sign-Off**                   | Final physical + timing verification                                                                   | DRC, LVS, and STA                                                                                          | Magic, Netgen, OpenSTA |

**Sign-off checks and supporting tools:**

| Task                           | Tool                            | Notes                                                                                |
| ------------------------------- | -------------------------------- | -------------------------------------------------------------------------------------- |
| DRC (Design Rule Check)         | **Magic VLSI**                   | Also used for antenna-rule checking and layout viewing                                 |
| LVS (Layout vs. Schematic)      | **Netgen** (+ Magic)             | Extracted SPICE (Magic) vs. Verilog netlist                                            |
| STA (Static Timing Analysis)    | **OpenSTA** (part of OpenROAD)   | Input: `.spef` (from DEF2SPEF RC extraction)                                           |
| Logic Equivalence Check (LEC)   | **Yosys**                        | Confirms function unchanged after netlist-modifying steps (CTS, post-placement opt)     |
| Design-for-Test (DFT)           | **Fault**                        | Scan insertion, ATPG, fault coverage/simulation                                        |
| GDSII viewing/streaming         | **KLayout** / **Magic**          | —                                                                                       |
| End-to-end orchestration        | **OpenLANE**                     | Wraps all of the above into one flow                                                   |

---

## 4. What is OpenLANE?

OpenLANE is the automated **RTL → GDSII** flow (aka automated Place-and-Route / Physical Implementation) that stitches together all the tools above.

- Originated from the **striVe** family of "open everything" SoCs (open PDK + open EDA + open RTL) — a true open-source tapeout experiment
- **Main goal:** produce a clean GDSII with no human intervention ("no-human-in-the-loop"), where clean = no LVS violations, no DRC violations (timing violations were still a work in progress at time of writing)
- Tuned for SkyWater 130nm (also supports XFAB180, GF130G)
- Containerized — functional out of the box
- Regression-tested against ~70 known designs to catch flow regressions

### Modes of operation

| Mode                               | Description                                                                                                |
| ------------------------------------ | ------------------------------------------------------------------------------------------------------------|
| **Autonomous**                     | "Push-button" — the full flow runs end-to-end automatically                                                 |
| **Interactive**                    | Run flow commands one-by-one — useful for debugging/tuning                                                   |
| **Design Space Exploration (DSE)** | Sweeps flow configurations to find the best settings for a given design (example configs across 43+ designs) |

### Running OpenLANE

- OpenLANE is Tcl-based and automates the flow command-by-command, or fully, from RTL to GDSII
- `package require openlane 0.9` loads the OpenLANE Tcl package
- Example design used in this workshop: **picorv32a** (one of OpenLANE's built-in examples), consisting of:
  - `picorv32a.v` – RTL source
  - `picorv32a.sdc` – design constraints
  - `sky130A_sky130_fd_sc_hd_config.tcl` – configuration file
- The `prep` step initializes the run directory before the flow begins

---

## 5. Two Concepts Every Beginner Trips On

### A. Antenna Rule Violations

During fabrication, long metal wire segments can act like unintended antennas: reactive ion etching accumulates charge on the wire, which can damage the connected transistor gate.

**Two fixes:**
1. **Bridging** — route through a higher metal layer as an intermediary (requires router awareness, not always available)
2. **Antenna diodes** — insert a diode cell to leak away the charge (provided by the standard cell library)

**OpenLANE's preventive approach:**
1. Insert a *fake* antenna diode next to every cell input after placement
2. Run the antenna checker (Magic) on the routed layout
3. If a violation is reported on a cell input pin, swap the fake diode for a real one

### B. Logic Equivalence Check (LEC)

Any time the netlist is modified post-synthesis (CTS insertion, post-placement optimization), you must re-verify functional equivalence. Yosys-based LEC does this by comparing pre/post-modification netlists formally — not just via simulation.

---

## 6. Quick-Start Mental Model (TL;DR)

1. Get comfortable with the RTL → GDSII pipeline order: `Synth → Floorplan/Power Plan → Place → CTS → Route → Sign-off`
2. Know which tool owns which stage (Section 3) — when something breaks, you'll know where to look
3. Understand the PDK is your contract with the fab — SKY130 is the standard free playground
4. Learn to read DRC / LVS / STA reports — these are your sign-off gates
5. Try OpenLANE in **interactive mode** first (not autonomous) so you can inspect each stage's output before trusting the push-button flow
6. Watch out for antenna violations and always re-run LEC after any netlist-altering step

---

## 7. Tools & Technologies Used

- **OpenLANE** – automated RTL-to-GDSII flow
- **OpenROAD** – open-source physical design tool suite
- **Yosys / ABC** – RTL synthesis
- **Magic / KLayout** – layout viewing, editing, DRC, antenna checks
- **Netgen** – LVS
- **OpenSTA** – static timing analysis
- **Fault** – design-for-test (DFT)
- **ngspice** – circuit simulation
- **SkyWater 130nm (sky130) PDK**

---

## Resources

- OpenLANE: [openlane.io](https://openlane.io)
- SkyWater open PDK: [github.com/google/skywater-pdk](https://github.com/google/skywater-pdk)
- OpenROAD Project
- Fault (DFT toolchain): bit.ly/3gSyIUU
- CloudV (cloud Verilog IDE/simulator): [cloudv.io](https://cloudv.io)

---

## Source

This README summarizes handwritten annotations and terminal screenshots from workshop slide deck `1_3_vsdworkshop.pdf`.
