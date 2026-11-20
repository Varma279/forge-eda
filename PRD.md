# ForgeEDA (`forge`) — Unified CLI & RL Physical Design Framework
**Document Type:** Master Engineering & Architecture Specification (For Claude Code Implementation)  
**Target Environments:** Linux (Ubuntu 20.04/22.04), Google Colab, Kaggle Notebooks  

---

## 1. Vision & Core Objectives
**ForgeEDA** is a unified, terminal-first open-source Electronic Design Automation (EDA) and AI physical design framework modeled after the usability of Ignytion IO’s Crucible (`igny`), accessible via the single CLI command `forge`.

### What It Solves
1. **Zero-Friction Unified Toolchain:** Users install and update Yosys, OpenROAD, OpenSTA, KLayout, NVIDIA AutoDMP/DREAMPlace, and NVIDIA INSTA from one CLI without manually compiling C++ codebases for weeks.
2. **Integrated Project & File Management:** Built-in workspace scaffolding (`forge init`), an interactive terminal file browser (`forge files`), direct tool launchers (`forge open <tool>`), and offline manual pages (`forge man`).
3. **Validated AI/RL Optimization Loop:** A reproducible placement and timing optimization engine that pairs GPU-accelerated placement (DREAMPlace/AutoDMP) and GPU timing (NVIDIA INSTA) with OpenROAD reference verification.
4. **Cloud-Notebook Ready:** Designed to run within the disk, RAM, and runtime constraints of Google Colab and Kaggle without requiring Docker daemons.

---

## 2. Scope & Engineering Boundaries
* **Primary Deliverable:** A working `forge` CLI capable of managing chip design workspaces, wrapping core EDA/AI tools cleanly, running reproducible baseline placement/timing evaluations on a reference design (`gcd` on an open PDK like `sky130` or `ASAP7`), and executing validated parameter optimization (Random Search -> Bayesian Optimization -> PPO RL).
* **Metric Honesty:** Until explicit post-route area and power extraction are verified, report results strictly as **Placement/QoR Optimization** (HPWL, congestion/overflow, and setup slack)—never claim full PPA signoff or manufacturing readiness.
* **Explicit Exclusions for v1.0:** Custom GNN policy training from scratch, multi-die/3D-IC flows, and unsupported proprietary PDKs.

---

## 3. CLI User Experience & Command Specification
Implement `forge` using **Typer** and **Rich** (plus **Textual** for the terminal UI) with the following exact command surface:

### 3.1 Installation, Environment & Self-Update Commands
* `forge setup [--profile full|cpu-mock|colab]` — Detects OS, Python, CUDA, and GPU availability; bootstraps isolated `micromamba` environments; installs pinned tool binaries and PDKs (`volare`); runs smoke tests.
* `forge doctor` — Prints a health-check table of installed tool versions, CUDA compatibility, disk usage, and environment paths.
* `forge update [tool_name|all]` — Updates ForgeEDA or individual wrapped backends (e.g., `forge update autodmp`, `forge update openroad`) against pinned compatibility manifests without breaking working environments.

### 3.2 Workspace & File Management Commands
* `forge init <project_name> [--pdk sky130|asap7]` — Scaffolds a standardized chip workspace (`rtl/`, `constraints/`, `configs/`, `runs/`, `checkpoints/`) with a starter `design.yaml` and sample RTL (`gcd`).
* `forge files [path]` — Launches a lightweight **Textual** terminal file manager allowing users to browse RTL, inspect `.def`/`.rpt` outputs, view run manifests, and manage files without leaving the terminal.
* `forge clean [--keep-best]` — Removes failed or temporary trial directories to reclaim Colab/Kaggle disk space while preserving the best feasible placement checkpoints.
* `forge export <archive_path>` — Packages run manifests, best `.def`/`.gds` artifacts, metrics, and checkpoints for persistent backup (e.g., to Google Drive or Kaggle `/kaggle/working`).

### 3.3 EDA Flow & AI Optimization Commands
* `forge synth` — Runs Yosys to synthesize RTL into a gate-level netlist.
* `forge place [--mode default|autodmp|rl]` — Executes placement via OpenROAD or GPU-accelerated DREAMPlace/AutoDMP and verifies placement legality.
* `forge sta [--engine openroad|insta]` — Runs reference static timing analysis (OpenROAD/OpenSTA with RC parasitics) or GPU timing (NVIDIA INSTA).
* `forge route` — Executes global/detailed routing and GDSII export via OpenROAD.
* `forge optimize [--method random|bayesian|ppo] [--budget N]` — Runs hyperparameter/RL placement optimization loops under a fixed evaluation budget.
* `forge run` — Executes the full configured end-to-end flow (`synth -> place -> sta -> route`) with step caching.

### 3.4 Direct Tool Access & Documentation
* `forge open <yosys|openroad|klayout|insta>` — Drops the user directly into the underlying tool's interactive shell/GUI with all PDK and project environment variables pre-loaded.
* `forge man [command]` — Renders Rich-formatted, searchable offline manual pages explaining every command, flag, configuration parameter, and example workflow.

---

## 4. Critical Precautions & Guardrails for Claude Code
**MANDATORY:** Follow these engineering rules strictly to avoid common failure modes when integrating research EDA tools:

1. **Inspect Upstream Code Before Writing Adapters (Zero Guessing):**
   * Do **not** invent CLI flags or output metric names. Inspect the actual checked-out repositories first.
   * *Example:* Upstream DREAMPlace is invoked via a **JSON configuration file**, not arbitrary CLI flags like `--wirelength_weight`. Write a JSON generator adapter matching the pinned schema.
2. **Isolate Conflicting Tool Environments:**
   * Do **not** force all tools into one monolithic Python environment if CUDA/PyTorch versions clash. Use separate `micromamba` sub-environments under `~/.forge/envs/` (e.g., `forge-core`, `forge-dreamplace`, `forge-insta`) invoked cleanly via subprocess wrappers.
   * Do **not** rely on Docker inside Colab or Kaggle.
3. **Expose Only Functional Action Parameters:**
   * Every parameter exposed to the optimizer or RL agent must map to a verified setting in the backend configuration. Never include dead/unused parameters (such as unimplemented macro halo arguments) in the action space.
4. **Strict Failure Handling — Never Fabricate Metrics:**
   * Every subprocess call must check exit codes, capture `stdout`/`stderr`, enforce strict timeouts, and verify that output files (e.g., placed `.def`) exist and pass legality checks.
   * If a run crashes, times out, or produces illegal geometry, log an explicit failure state (`status: FAILED`) and assign a penalty—**never substitute dummy HPWL or zero slack values**.
5. **Valid Reference Timing & RC Setup:**
   * Before extracting timing metrics in OpenROAD, always load layer/wire RC parasitics and run `estimate_parasitics -global_routing`. Explicitly record clock periods, corners, and time units (`ns` vs `ps`) in every metric JSON.
6. **NVIDIA INSTA Integration & Licensing Gate:**
   * **License Check:** NVIDIA INSTA is distributed under the *NVIDIA Source Code License* (source-available, not standard OSI open source). Display a clear license notice during `forge setup` / `forge open insta`.
   * **Validation Before RL:** First reproduce INSTA's upstream testcase. Build an explicit timing-data converter (converting DEF/SPEF/SDC/Liberty data into INSTA's exact expected schema) and benchmark INSTA's slack predictions against OpenROAD reference timing across multiple placements. Only allow INSTA inside the RL reward loop once rank-order correlation is verified.
   * Do **not** backpropagate INSTA timing gradients directly into macro coordinates without a mathematically validated differentiable coordinate bridge.
7. **Staged AI Progression (Baselines First):**
   * Do **not** implement PPO reinforcement learning until: (a) deterministic baseline runs pass, (b) Random Search runs pass, and (c) the Gymnasium environment passes `stable_baselines3.common.env_checker.check_env` using both a fast CPU mock backend and real tool execution.
   * Clearly distinguish single-step black-box parameter tuning (suited for Random/Bayesian search via Optuna) from multi-step sequential state transitions (suited for PPO).

---

## 5. System Architecture & Repository Layout

```text
forge-eda/
├── pyproject.toml                 # Package metadata, dependencies, `forge` CLI entry point
├── CLAUDE.md                      # Quick-reference build rules & test commands for Claude Code
├── PRD.md                         # Master product & technical requirements
├── forge/
│   ├── __init__.py
│   ├── cli.py                     # Typer CLI commands (setup, init, files, place, sta, optimize, man, update)
│   ├── config.py                  # Pydantic schemas for design.yaml, tool manifests, and trial configs
│   ├── installer.py               # Micromamba + Volare PDK + tool environment manager & updater
│   ├── file_manager.py            # Textual TUI file & run explorer (`forge files`)
│   ├── man_pages.py               # Rich offline documentation system (`forge man`)
│   ├── adapters/                  # Isolated subprocess wrappers for each backend
│   │   ├── base.py                # Timeout, return-code, log capture, and artifact validation logic
│   │   ├── yosys_adapter.py       # RTL synthesis wrapper
│   │   ├── dreamplace_adapter.py  # JSON config writer + DREAMPlace/AutoDMP runner
│   │   ├── openroad_adapter.py    # Floorplan, legality check, RC parasitics, routing, reference STA
│   │   └── insta_adapter.py       # Timing data converter + NVIDIA INSTA GPU STA runner
│   ├── metrics/
│   │   ├── parsers.py             # Tested regex/JSON parsers for OpenROAD, DREAMPlace, INSTA logs
│   │   ├── validator.py           # Hard feasibility & DEF legality checks
│   │   └── reward.py              # Baseline-normalized QoR reward calculation
│   └── rl/
│       ├── env.py                 # Gymnasium-compliant environment (seeded reset/step, truncation vs termination)
│       └── optimizers.py          # Baseline runner, Random Search, Optuna Bayesian Search, SB3 PPO
├── configs/
│   ├── toolchain_manifest.yaml    # Pinned Git commits, Conda/PyPI versions, and compatibility matrix
│   └── rewards.yaml               # Default normalization scales and objective weights
├── templates/
│   └── gcd_sky130/                # Ready-to-run starter bundle (Verilog, SDC, design.yaml)
└── tests/
    ├── fixtures/                  # Sample success/failure logs & mock DEF files
    ├── test_parsers.py            # Unit tests for log parsers & failure detection
    ├── test_adapters_mock.py      # Fast CPU-only mock adapter tests
    └── test_gym_env.py            # SB3 `check_env` & determinism verification
```

---

## 6. Standardized Project Workspace (Created by `forge init`)
When a user runs `forge init my_design`, ForgeEDA creates a self-contained, organized directory:

```text
my_design/
├── design.yaml          # Unified project config (top module, clock period, PDK, backend engines)
├── rtl/                 # User Verilog / SystemVerilog files
├── constraints/         # SDC timing constraint files
├── runs/                # Unique, timestamped trial folders (never overwritten)
│   └── trial_0001/
│       ├── config.json  # Exact parameters & random seed used
│       ├── status.json  # PASS/FAIL, failure reason, wall-clock runtime, tool versions
│       ├── metrics.json # Unit-labeled HPWL, overflow, WNS/TNS, normalized reward
│       ├── logs/        # Preserved stdout/stderr from every tool
│       └── artifacts/   # Synthesized netlist, placed .def, routed .gds
└── checkpoints/         # Best feasible candidate DEF + optimizer/RL model weights
```

---

## 7. Normalization & Reward Formulation
All optimization loops must enforce **hard feasibility checks** before computing a reward. Any candidate failing DEF legality, routing convergence, or timing validity receives `feasible = false` and cannot be ranked as a best solution.

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
* **Gate 2 — Real EDA Flow & Deterministic Baseline:**
  * Install Yosys, OpenROAD, OpenSTA, and Volare PDK (`sky130`); implement adapters and run `forge run` on the `gcd` starter design.
  * *Acceptance:* Generates valid, legal `.def`, configures RC parasitics, and logs finite, unit-labeled baseline HPWL, congestion, and WNS/TNS.
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
* **OpenROAD-flow-scripts (ORFS):** https://github.com/the-openroad-project/openroad-flow-scripts
* **ORFS Documentation:** https://openroad-flow-scripts.readthedocs.io/en/latest/index2.html
* **OpenROAD Engine Docs:** https://openroad.readthedocs.io/en/latest/
* **OpenROAD RC / Parasitics Estimation:** https://openroad.readthedocs.io/en/latest/main/openroad/est/README.html
* **NVIDIA AutoDMP:** https://github.com/NVlabs/AutoDMP
* **DREAMPlace Build & JSON Config Docs:** https://github.com/NVlabs/AutoDMP/blob/main/README_DREAMPlace.md
* **NVIDIA INSTA (GPU Differentiable STA):** https://github.com/NVlabs/INSTA
* **Volare (Open PDK Manager):** https://github.com/efabless/volare
* **Gymnasium (RL Environment Standard):** https://gymnasium.farama.org/
* **Stable-Baselines3 (`check_env` & PPO):** https://stable-baselines3.readthedocs.io/en/master/guide/custom_env.html
* **Optuna (Bayesian Optimization):** https://optuna.readthedocs.io/
* **Typer & Rich (CLI & Man Pages):** https://typer.tiangolo.com/
* **Textual (Terminal File Manager):** https://textual.textualize.io/
* **Google Colab Resource FAQ:** https://research.google.com/colaboratory/intl/en-GB/faq.html
