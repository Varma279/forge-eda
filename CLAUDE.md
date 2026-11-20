# CLAUDE.md — Engineering Rules for ForgeEDA (`forge`)

## Project Overview
You are building **ForgeEDA** (`forge`), an open-source, terminal-first EDA and AI physical design CLI framework compatible with Linux, Google Colab, and Kaggle. Full requirements (including digital, analog/mixed-signal, memory compilers, DRC/LVS tools, and SKY130/GF180MCU/SG13G2 PDKs) are in `PRD.md`.

## Mandatory Engineering Guardrails
1. **Never Guess Upstream APIs:** Before writing any wrapper for OpenROAD, DREAMPlace/AutoDMP, NVIDIA INSTA, Magic, Netgen, OpenRAM, or gdsfactory, inspect the upstream repository docs/READMEs. DREAMPlace uses a JSON configuration file—do not invent CLI flags.
2. **No Docker Required & Modular Disk Profiles:** Target `micromamba` isolated environments (`~/.forge/envs/`) and `volare`/official PDK installers so everything runs smoothly on Google Colab and Kaggle without exceeding disk quotas. Keep GUI tools (`klayout`, `magic`, `xschem`, `gtkwave`) strictly headless during automated runs.
3. **Never Fabricate Metrics:** Every subprocess adapter must verify return codes, capture `stdout`/`stderr`, enforce timeouts, and validate output files (`.def`, `.gds`, DRC/LVS reports). Failed runs must record `status: FAILED` and never inject dummy HPWL, zero slack, or fake DRC/LVS passes.
4. **Reference Timing & RC Setup:** Always configure wire/layer RC parasitics and run `estimate_parasitics -global_routing` in OpenROAD before extracting timing metrics. Record explicit units (`ns` vs `ps`).
5. **NVIDIA INSTA Gate:** Treat INSTA as source-available (NVIDIA Source Code License). Reproduce its upstream testcase first and validate rank-order correlation against OpenROAD timing before allowing `--engine insta` in optimization rewards.
6. **Gate-by-Gate Execution:** Complete and test each Gate in `PRD.md` Section 8 sequentially. Do not implement PPO until Gate 1–4 pass and `stable_baselines3.common.env_checker.check_env` succeeds.

## Development & Test Commands
* Install package in editable mode: `pip install -e ".[dev]"`
* Run fast CPU mock test suite: `pytest tests/test_parsers.py tests/test_adapters_mock.py tests/test_gym_env.py`
* Verify CLI entry point: `forge --help`, `forge man`, `forge doctor`
