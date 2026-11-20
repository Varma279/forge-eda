# ForgeEDA (`forge`) — Unified CLI & RL Physical Design Framework
**Document Type:** Master Engineering & Architecture Specification (v1.2 — Full Digital, Analog/Mixed-Signal & Memory Ecosystem)  
**Target Environments:** Linux (Ubuntu 20.04/22.04), Google Colab, Kaggle Notebooks  

---

## 1. Vision & Core Objectives
**ForgeEDA** is a unified, terminal-first open-source Electronic Design Automation (EDA) and AI physical design framework modeled after the usability of Ignytion IO’s Crucible (`igny`), accessible via the single CLI command `forge`.

### What It Solves
1. **Complete Silicon Design Suite in One CLI:** Bundles digital RTL simulation/synthesis/P&R, GPU-accelerated AI placement/timing, memory compilers, analog/mixed-signal schematic & SPICE tools, and DRC/LVS signoff verification—eliminating manual multi-week C++ compilations.
2. **Modular Profiles to Save Disk & Time:** Installs lightweight core profiles by default (`cpu-mock`, `colab`, `digital`, `analog`, `full`) via isolated `micromamba` environments so cloud notebooks (Colab/Kaggle) never run out of disk space.
3. **Integrated Project & File Management:** Built-in workspace scaffolding (`forge init`), an interactive terminal file browser (`forge files`), direct tool launchers (`forge open <tool>`), and offline manual pages (`forge man`).
4. **Validated AI/RL Optimization Loop:** A reproducible placement and timing optimization engine pairing GPU placement (DREAMPlace/AutoDMP) and GPU timing (NVIDIA INSTA) with OpenROAD reference verification.

---

## 2. Complete Integrated Tool & PDK Catalog

### 2.1 Digital RTL Simulation & Verification
* **Icarus Verilog (`iverilog`):** Event-driven Verilog simulation and testbench execution.
* **Verilator (`verilator`):** High-speed cycle-accurate Verilog/SystemVerilog-to-C++ simulator and linter.
* **GTKWave (`gtkwave`):** Waveform viewer for `.vcd`/`.fst` simulation dumps (plus headless waveform summary checks for Colab/Kaggle).

### 2.2 Digital RTL-to-GDSII Flow
* **Yosys:** RTL logic synthesis (`Verilog -> Gate-Level Netlist`).
* **OpenROAD:** Floorplanning, PDN generation, Clock Tree Synthesis (CTS), global/detailed routing, and GDSII export.
* **OpenSTA:** Reference CPU Static Timing Analysis (STA) and RC parasitic estimation.

### 2.3 GPU-Accelerated AI Physical Design & RL Engines
* **DREAMPlace:** Underlying GPU-accelerated analytical placement engine.
* **NVIDIA AutoDMP:** Automated concurrent macro and standard-cell placement optimizer.
* **NVIDIA INSTA:** GPU-accelerated differentiable static timing analysis engine *(source-available under the NVIDIA Source Code License)*.
* **Gymnasium, Stable-Baselines3 (PPO), Optuna, PyTorch:** Standardized RL environment, Bayesian optimization, and GPU tensor runtime.

### 2.4 Memory Compilers & Python Layout Automation
* **OpenRAM:** Open-source static random-access memory (SRAM) compiler generating LEF, GDSII, Liberty (`.lib`), and Verilog models.
* **DFFRAM:** Standard-cell-based RAM compiler compatible with OpenROAD digital flows.
* **gdsfactory:** Python-driven programmatic layout, photonic/analog/digital parametric cell (PCell) generation, and GDSII manipulation.

### 2.5 Analog, Mixed-Signal & Physical Signoff Verification
* **Xschem:** Schematic capture and netlist generation for analog and mixed-signal circuits.
* **ngspice:** Open-source mixed-level/mixed-signal SPICE circuit simulator.
* **Magic VLSI (`magic`):** Interactive/batch VLSI layout editor, parasitic extraction (`ext2spice`), and Design Rule Checking (DRC).
* **KLayout (`klayout`):** High-speed GDSII/OASIS layout viewer, DRC/LVS engine, and headless layout PNG/SVG renderer (ideal for Colab/Kaggle visualization).
* **Netgen (`netgen`):** Layout Versus Schematic (LVS) netlist comparison tool verifying extracted layout netlists against schematic/gate-level netlists.

### 2.6 Supported Open-Source Process Design Kits (PDKs)
Managed and version-pinned via **Volare** (plus IHP official PDK scripts where required) without downloading unnecessary multi-gigabyte archives:
* **SkyWater `sky130` (130nm CMOS):** Backed by Google and SkyWater; default PDK for digital, mixed-signal, and educational/hobbyist tapeouts.
* **GlobalFoundries `gf180mcu` (180nm CMOS):** Open-source PDK suited for mixed-signal and 5V tolerant designs.
* **IHP `sg13g2` (130nm BiCMOS):** Open-source BiCMOS PDK geared toward high-frequency RF, analog, and millimeter-wave designs.
* **Predictive `asap7` (7nm FinFET):** Optional predictive research PDK for testing advanced-node timing features in OpenROAD and NVIDIA INSTA.

---

## 3. CLI User Experience & Command Specification
Implement `forge` using **Typer** and **Rich** (plus **Textual** for the terminal UI):

### 3.1 Installation, Environment & Self-Update Commands
* `forge setup [--profile cpu-mock|colab|digital|analog|full]` — Detects OS, Python, CUDA, and GPU availability; bootstraps isolated `micromamba` environments; installs pinned tool binaries and PDKs; runs smoke tests.
  * *Profile Disk Protection:* `--profile colab` installs the lean digital + GPU AI stack (`yosys`, `openroad`, `opensta`, `iverilog`, `verilator`, `magic`, `netgen`, `klayout`, `gdsfactory`, `autodmp`, `insta`, `sky130`) keeping disk usage compact. `--profile full` adds `xschem`, `ngspice`, `gtkwave`, `openram`, `dffram`, `gf180mcu`, and `sg13g2`.
* `forge doctor` — Prints a health-check table of all installed tool versions, active PDKs, CUDA compatibility, and disk usage.
* `forge update [tool_name|pdk_name|all]` — Updates ForgeEDA or individual wrapped backends (e.g., `forge update openroad`, `forge update gdsfactory`, `forge update pdk`) against `configs/toolchain_manifest.yaml`.

### 3.2 Workspace & File Management Commands
* `forge init <project_name> [--pdk sky130|gf180mcu|sg13g2|asap7] [--type digital|mixed-signal]` — Scaffolds a standardized chip workspace (`rtl/`, `analog/`, `macros/`, `constraints/`, `configs/`, `runs/`, `checkpoints/`) with a starter `design.yaml` and sample design.
* `forge files [path]` — Launches a lightweight **Textual** terminal file manager to browse RTL, schematics, `.def`/`.gds` layouts, waveforms, and run manifests without leaving the terminal.
* `forge clean [--keep-best]` — Removes failed or temporary trial directories to reclaim Colab/Kaggle disk space while preserving the best feasible checkpoints.
* `forge export <archive_path>` — Packages run manifests, best `.def`/`.gds` artifacts, metrics, and checkpoints for persistent backup (e.g., to Google Drive or Kaggle `/kaggle/working`).

### 3.3 Simulation, Memory, Analog & RTL-to-GDSII Flow Commands
* `forge sim [--tool iverilog|verilator] [--wave]` — Compiles and runs RTL testbenches, generating `.vcd`/`.fst` waveforms (and optionally opening `gtkwave` on desktop Linux).
* `forge spice <netlist.spice>` — Runs analog/mixed-signal circuit simulation via `ngspice`.
* `forge macro [--compiler openram|dffram] --config <ram_cfg.py>` — Generates hardened SRAM/DFFRAM macros (`.lef`, `.gds`, `.lib`, `.v`) and registers them into `macros/`.
* `forge synth` — Runs Yosys to synthesize RTL into a gate-level netlist.
* `forge place [--mode default|autodmp|rl]` — Executes placement via OpenROAD or GPU-accelerated DREAMPlace/AutoDMP and verifies placement legality.
* `forge sta [--engine openroad|insta]` — Runs reference static timing analysis (OpenROAD/OpenSTA with RC parasitics) or GPU timing (NVIDIA INSTA).
* `forge route` — Executes global/detailed routing and GDSII export via OpenROAD (or programmatic layout assembly via `gdsfactory`).
* `forge verify [--drc] [--lvs]` — Runs headless DRC (`magic` / `klayout`) and LVS (`netgen` comparing extracted layout SPICE against reference netlist).
* `forge optimize [--method random|bayesian|ppo] [--budget N]` — Runs hyperparameter/RL placement optimization loops under a fixed evaluation budget.
* `forge run` — Executes the full configured end-to-end flow (`sim -> synth -> place -> sta -> route -> verify`) with step caching.

### 3.4 Direct Tool Access & Documentation
* `forge open <tool>` — Drops the user directly into any wrapped tool (`yosys`, `openroad`, `klayout`, `magic`, `xschem`, `ngspice`, `gtkwave`, `iverilog`, `verilator`, `netgen`, `openram`, `gdsfactory`, `insta`) with `PDK_ROOT` and `PDK` environment variables pre-loaded.
* `forge man [command|tool]` — Renders Rich-formatted, searchable offline manual pages explaining every command, wrapped tool, PDK setup, and example workflow.

---

## 4. Critical Precautions & Guardrails for Claude Code
**MANDATORY:** Follow these engineering rules strictly to avoid build failures or invalid benchmarks:

1. **Inspect Upstream Code Before Writing Adapters (Zero Guessing):**
   * Do **not** invent CLI flags or output metric names. Inspect checked-out repositories first.
   * *Example:* Upstream DREAMPlace is invoked via a **JSON configuration file**, not arbitrary CLI flags. Write a JSON generator adapter matching the pinned schema.
2. **Isolate Conflicting Tool Environments:**
   * Do **not** force all 25+ tools into one monolithic Python environment if CUDA/PyTorch/Tcl versions clash. Use separate `micromamba` sub-environments under `~/.forge/envs/` (e.g., `forge-core` for OpenROAD/Yosys/Magic/Netgen/Xschem/ngspice/iverilog/verilator/gdsfactory, `forge-dreamplace`, and `forge-insta`) invoked cleanly via subprocess wrappers.
   * Do **not** rely on Docker inside Colab or Kaggle.
3. **Headless Compatibility for GUI Tools on Colab/Kaggle:**
   * Tools like `klayout`, `magic`, `xschem`, and `gtkwave` have GUI modes that crash without an X11 display (`DISPLAY` unset). Always run them in batch/headless CLI mode (`-b` / `-noconsole` / `-n`) during automated flows, and reserve interactive GUI launch strictly for `forge open <tool>` when a display is present.
4. **Expose Only Functional Action Parameters:**
   * Every parameter exposed to the optimizer or RL agent must map to a verified setting in the backend configuration. Never include dead/unused parameters in the action space.
5. **Strict Failure Handling — Never Fabricate Metrics:**
   * Every subprocess call must check exit codes, capture `stdout`/`stderr`, enforce strict timeouts, and verify that output files exist and pass legality/DRC/LVS checks.
   * If a run crashes, times out, or fails verification, log `status: FAILED` and assign a penalty—**never substitute dummy HPWL, zero slack, or fake DRC pass counts**.
6. **Valid Reference Timing & RC Setup:**
   * Before extracting timing metrics in OpenROAD, always load layer/wire RC parasitics and run `estimate_parasitics -global_routing`. Explicitly record clock periods, corners, and time units (`ns` vs `ps`) in every metric JSON.
7. **NVIDIA INSTA Integration & Licensing Gate:**
   * **License Check:** NVIDIA INSTA is distributed under the *NVIDIA Source Code License* (source-available, not standard OSI open source). Display a clear license notice during `forge setup` / `forge open insta`.
   * **Validation Before RL:** Reproduce INSTA's upstream testcase first, build an explicit timing-data converter, and benchmark INSTA slack predictions against OpenROAD reference timing across >= 5 candidate placements before enabling `--engine insta` in RL rewards.
8. **Staged AI Progression (Baselines First):**
   * Do **not** implement PPO reinforcement learning until: (a) deterministic baseline runs pass, (b) Random Search runs pass, and (c) the Gymnasium environment passes `stable_baselines3.common.env_checker.check_env` using both a fast CPU mock backend and real tool execution.

---

## 5. System Architecture & Repository Layout

```text
forge-eda/
├── pyproject.toml                 # Package metadata, dependencies, `forge` CLI entry point
├── CLAUDE.md                      # Quick-reference build rules & test commands for Claude Code
├── PRD.md                         # Master product & technical requirements (v1.2)
├── forge/
│   ├── __init__.py
│   ├── cli.py                     # Typer CLI commands (setup, init, files, sim, spice, macro, synth, place, sta, route, verify, optimize, man, update)
│   ├── config.py                  # Pydantic schemas for design.yaml, PDK configs, and trial configs
│   ├── installer.py               # Micromamba + Volare/IHP PDK + tool environment manager & updater
│   ├── file_manager.py            # Textual TUI file & run explorer (`forge files`)
│   ├── man_pages.py               # Rich offline documentation system (`forge man`)
│   ├── adapters/                  # Isolated subprocess wrappers for each backend
│   │   ├── base.py                # Timeout, return-code, log capture, and artifact validation logic
│   │   ├── sim_adapter.py         # Icarus Verilog, Verilator & GTKWave wrapper
│   │   ├── analog_adapter.py      # Xschem & ngspice wrapper
│   │   ├── macro_adapter.py       # OpenRAM & DFFRAM memory compiler wrapper
│   │   ├── yosys_adapter.py       # RTL synthesis wrapper
│   │   ├── dreamplace_adapter.py  # JSON config writer + DREAMPlace/AutoDMP runner
│   │   ├── openroad_adapter.py    # Floorplan, legality check, RC parasitics, routing, reference STA
│   │   ├── insta_adapter.py       # Timing data converter + NVIDIA INSTA GPU STA runner
│   │   └── signoff_adapter.py     # Magic VLSI, KLayout (DRC/rendering), Netgen (LVS) & gdsfactory wrapper
│   ├── metrics/
│   │   ├── parsers.py             # Tested regex/JSON parsers for OpenROAD, DREAMPlace, INSTA, DRC/LVS logs
│   │   ├── validator.py           # Hard feasibility, DEF legality, and DRC/LVS checks
│   │   └── reward.py              # Baseline-normalized QoR reward calculation
│   └── rl/
│       ├── env.py                 # Gymnasium-compliant environment (seeded reset/step, truncation vs termination)
│       └── optimizers.py          # Baseline runner, Random Search, Optuna Bayesian Search, SB3 PPO
├── configs/
│   ├── toolchain_manifest.yaml    # Pinned Git commits, Conda/PyPI versions, and PDK hashes
│   └── rewards.yaml               # Default normalization scales and objective weights
├── templates/
│   ├── gcd_sky130/                # Ready-to-run digital starter bundle (Verilog, testbench, SDC, design.yaml)
│   └── inverter_sg13g2/           # Starter analog/mixed-signal Xschem/ngspice bundle
└── tests/
    ├── fixtures/                  # Sample success/failure logs, DRC/LVS reports & mock DEF files
    ├── test_parsers.py            # Unit tests for log parsers & failure detection
    ├── test_adapters_mock.py      # Fast CPU-only mock adapter tests
    └── test_gym_env.py            # SB3 `check_env` & determinism verification
```

---

## 6. Standardized Project Workspace (Created by `forge init`)

```text
my_design/
├── design.yaml          # Unified project config (top module, clock period, PDK: sky130/gf180mcu/sg13g2/asap7)
├── rtl/                 # Verilog / SystemVerilog source & testbench files
├── analog/              # Xschem schematics (.sch) & ngspice testbenches (.spice)
├── macros/              # OpenRAM / DFFRAM generated SRAM blocks (.lef, .gds, .lib, .v)
├── constraints/         # SDC timing constraint files
├── runs/                # Unique, timestamped trial folders (never overwritten)
│   └── trial_0001/
│       ├── config.json  # Exact parameters & random seed used
│       ├── status.json  # PASS/FAIL, failure reason, wall-clock runtime, tool versions
│       ├── metrics.json # Unit-labeled HPWL, overflow, WNS/TNS, DRC/LVS status, normalized reward
│       ├── logs/        # Preserved stdout/stderr from every tool
│       └── artifacts/   # Waveforms (.vcd), netlists, placed .def, routed .gds, DRC/LVS reports
└── checkpoints/         # Best feasible candidate DEF/GDS + optimizer/RL model weights
```

---

## 7. Normalization & Reward Formulation
All optimization loops must enforce **hard feasibility checks** before computing a reward. Any candidate failing DEF legality, routing convergence, timing validity, or DRC/LVS checks receives `feasible = false` and cannot be ranked as a best solution.

For feasible designs, compute the baseline-normalized placement QoR reward:

$$R = -\left[ w_1 \left(rac{	ext{HPWL}}{	ext{HPWL}_0}ight) + w_2 \left(rac{	ext{Overflow}}{	ext{Overflow}_{	ext{scale}}}ight) + w_3 \left(rac{\max(0, -	ext{WNS})}{T_{	ext{scale}}}ight) ight]$$

* $	ext{HPWL}_0$: Measured HPWL from the unoptimized deterministic baseline run.
* $	ext{Overflow}_{	ext{scale}}, T_{	ext{scale}}$: Documented normalization constants recorded in `configs/rewards.yaml`.
* Always retain the **best feasible candidate** across all trials (`checkpoints/best_feasible.def`), independent of the final RL policy weights.

---

## 8. Phased Implementation Plan & Acceptance Gates
Execute development sequentially and pass each gate before moving to the next:

* **Gate 1 — CLI Skeleton, File Manager & Toolchain Setup:**
  * Implement `forge init`, `forge files`, `forge man`, `forge doctor`, and `forge setup --profile cpu-mock`.
  * *Acceptance:* All CLI commands execute cleanly; `forge init` creates a valid workspace; unit tests pass on mock fixtures.
* **Gate 2 — Core Open-Source EDA Flow, PDKs & Deterministic Baseline:**
  * Install `yosys`, `openroad`, `opensta`, `iverilog`, `verilator`, `gtkwave`, `magic`, `netgen`, `klayout`, `gdsfactory`, `xschem`, `ngspice`, `openram`/`dffram`, and PDKs (`sky130`, `gf180mcu`, `sg13g2`) via `micromamba`/`volare`.
  * *Acceptance:* `forge sim`, `forge synth`, `forge run`, and `forge verify --drc --lvs` pass on the `gcd` starter design with legal `.def`, RC parasitics, and unit-labeled baseline metrics.
* **Gate 3 — GPU Placement Integration (DREAMPlace / AutoDMP):**
  * Install pinned DREAMPlace/AutoDMP in an isolated environment; implement the JSON config adapter.
  * *Acceptance:* Changing action parameters demonstrably modifies the generated DREAMPlace JSON and alters output placement metrics; forced errors/timeouts are caught without fake metrics.
* **Gate 4 — NVIDIA INSTA Timing Validation:**
  * Install INSTA in an isolated environment; reproduce upstream testcase; implement `insta_adapter.py` to convert placement/timing data.
  * *Acceptance:* Compare INSTA slack outputs against OpenROAD reference timing across >= 5 candidate placements with aligned clocks/units; log correlation report before enabling `--engine insta`.
* **Gate 5 — Optimization & RL Loop:**
  * Implement Gymnasium environment (`forge/rl/env.py`), pass SB3 `check_env`, and benchmark Random Search vs. Optuna Bayesian Optimization vs. PPO under equal evaluation budgets.
  * *Acceptance:* Reproducible seeded runs, checkpoint resume (`forge export` / reload), and comparative summary report.

---

## 9. Verified Official Reference Links
* **OpenROAD & ORFS:** https://github.com/the-openroad-project/openroad-flow-scripts | https://openroad.readthedocs.io/en/latest/
* **Yosys HQ:** https://github.com/YosysHQ/yosys
* **NVIDIA AutoDMP & DREAMPlace:** https://github.com/NVlabs/AutoDMP | https://github.com/NVlabs/AutoDMP/blob/main/README_DREAMPlace.md
* **NVIDIA INSTA:** https://github.com/NVlabs/INSTA
* **Magic VLSI & Netgen (Open Circuit Design):** http://opencircuitdesign.com/magic/ | http://opencircuitdesign.com/netgen/
* **KLayout & gdsfactory:** https://www.klayout.de/ | https://github.com/gdsfactory/gdsfactory
* **Icarus Verilog, Verilator & GTKWave:** https://github.com/steveicarus/iverilog | https://verilator.org/ | https://github.com/gtkwave/gtkwave
* **OpenRAM & DFFRAM:** https://github.com/VLSIDA/OpenRAM | https://github.com/AUCOHL/DFFRAM
* **Xschem & ngspice:** https://xschem.sourceforge.io/stefan/index.html | https://ngspice.sourceforge.io/
* **Open PDKs (Volare, SkyWater SKY130, GF180MCU, IHP SG13G2):** https://github.com/efabless/volare | https://github.com/google/skywater-pdk | https://github.com/google/gf180mcu-pdk | https://github.com/IHP-GmbH/IHP-Open-PDK
* **Gymnasium, Stable-Baselines3 & Optuna:** https://gymnasium.farama.org/ | https://stable-baselines3.readthedocs.io/en/master/guide/custom_env.html | https://optuna.readthedocs.io/
* **Typer, Rich & Textual:** https://typer.tiangolo.com/ | https://textual.textualize.io/
