# CLAUDE.md — Engineering Rules for ForgeEDA (`forge`)

## Project Overview
You are building **ForgeEDA** (`forge`), an open-source, terminal-first EDA and AI physical design CLI framework compatible with Linux, Google Colab, and Kaggle. Full requirements are in `PRD.md`.

## Mandatory Engineering Guardrails
1. **Never Guess Upstream APIs:** Before writing any wrapper for OpenROAD, DREAMPlace/AutoDMP, or NVIDIA INSTA, inspect the upstream repository docs/READMEs. DREAMPlace uses a JSON configuration file—do not invent CLI flags.
2. **No Docker Required:** Target `micromamba` isolated environments (`~/.forge/envs/`) and `volare` for PDKs so everything runs smoothly on Google Colab and Kaggle.
3. **Never Fabricate Metrics:** Every subprocess adapter must verify return codes, capture `stdout`/`stderr`, enforce timeouts, and validate output files (`.def`, reports). Failed runs must record `status: FAILED` and never inject dummy HPWL or zero slack.
4. **Reference Timing & RC Setup:** Always configure wire/layer RC parasitics and run `estimate_parasitics -global_routing` in OpenROAD before extracting timing metrics. Record units (`ns` or `ps`).
5. **NVIDIA INSTA Gate:** Treat INSTA as source-available (NVIDIA Source Code License). Reproduce its upstream testcase first and validate rank-order correlation against OpenROAD timing before allowing `--engine insta` in optimization rewards.
6. **Gate-by-Gate Execution:** Complete and test each Gate in `PRD.md` Section 8 sequentially. Do not implement PPO until Gate 1–4 pass and `stable_baselines3.common.env_checker.check_env` succeeds.

## Development & Test Commands
* Install package in editable mode: `pip install -e ".[dev]"`
* Run fast CPU mock test suite: `pytest tests/test_parsers.py tests/test_adapters_mock.py tests/test_gym_env.py`
* Verify CLI entry point: `forge --help`, `forge man`, `forge doctor`
