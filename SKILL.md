---
name: personal-engineering-style
description: Apply the user's personal engineering code generation style for Python, YAML, Bash/Shell, JSON/JSONL, TOML, Markdown, requirements.txt, and common ML/robotics/model inference project files. Use when the user explicitly invokes $personal-engineering-style, asks to generate or modify Python + YAML + Bash engineering code, requests training/evaluation/inference scripts, robotics or embodied AI model code, project scaffolding, config files, launch scripts, README docs, or asks for code to follow their personal engineering conventions.
---

# Personal Engineering Style

Use this skill to generate or modify engineering code in the user's preferred style.

Core preference:

> Keep the external project structure clear and reproducible, while keeping Python task code linear, lightweight, and easy to inspect.

## Priority

Follow this order:

```text
Current explicit user request > this personal engineering style > default AI style
```

If the request conflicts with this skill, satisfy the request and briefly note the conflict.

## Default Shape

Prefer:

- `src/` package structure for long-lived projects.
- `configs/` for YAML.
- `scripts/` for Bash launch scripts.
- `outputs/` for predictions, metrics, and logs.
- `requirements.txt` for dependencies.
- README with layout, input format, output format, config fields, and run commands.

Avoid by default:

- Makefile, Dockerfile, `.env`.
- Heavy config frameworks.
- Dataclass/pydantic config objects unless explicitly useful.
- Pipeline/Manager/Factory/Registry abstractions for simple tasks.
- Excessive OOP.

## Python Defaults

Use:

- `argparse` with subcommands such as `train`, `eval`, `infer`.
- `pathlib.Path` for all paths.
- Standard `logging` with simple format.
- Simple type hints on key functions.
- A class for naturally stateful model/policy/environment objects.
- Linear command handlers, especially for inference scripts.
- `tqdm` for long loops.

Do not default to:

- Module-level config loading or model execution.
- Global mutable state except a module-level `logger`.
- Dataset/Writer/Pipeline classes unless the project needs replaceable components.
- Complex generics or strict full-project typing.

Recommended config access pattern:

```python
cfg = load_yaml(config_path)
model_cfg = cfg["model"]
dataset_cfg = cfg["dataset"]
runtime_cfg = cfg["runtime"]
inference_cfg = cfg["inference"]
```

Recommended path rule:

```python
def resolve_config_path(config_path: Path, path_value: str) -> Path:
    path = Path(path_value)
    return path if path.is_absolute() else config_path.parent / path
```

YAML-relative paths are resolved from the config file's directory, not from the current working directory.

## YAML Defaults

Use YAML as the main human-written ML/robotics config format.

Prefer:

- `snake_case` keys.
- Clear names such as `checkpoint_path`, `input_path`, `prediction_path`, `batch_size`, `score_threshold`.
- Top-level groups: `model`, `dataset`, `train`, `eval`, `inference`, `runtime`.
- `true` / `false` for booleans.
- `null` for missing optional values.
- Few comments, mainly for global conventions.

Avoid:

- Flat configs for non-trivial projects.
- Abbreviations such as `ckpt`, `bs`, `thr`, `pred`.
- JSON or TOML as the main experiment config unless requested.

## Bash Defaults

Launch scripts should be strict, explicit, and log-saving:

```bash
#!/usr/bin/env bash
set -euo pipefail

PROJECT_ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
CONFIG_PATH="${CONFIG_PATH:-${PROJECT_ROOT}/configs/inference.yaml}"
CUDA_VISIBLE_DEVICES="${CUDA_VISIBLE_DEVICES:-0}"
DEVICE="${DEVICE:-cuda:0}"
LOG_DIR="${PROJECT_ROOT}/outputs/logs"

export CUDA_VISIBLE_DEVICES
export PYTHONPATH="${PROJECT_ROOT}/src:${PYTHONPATH:-}"

mkdir -p "${LOG_DIR}"

python -m robot_infer.cli infer \
  --config "${CONFIG_PATH}" \
  --device "${DEVICE}" \
  2>&1 | tee "${LOG_DIR}/inference.log"
```

Rules:

- Quote variables.
- Use one argument per line for commands with more than two arguments.
- Let Bash read environment overrides and pass them explicitly as CLI args.
- Python should not silently read environment variables by default.
- Manage both `CUDA_VISIBLE_DEVICES` and Python `--device` for GPU tasks.

## JSON / TOML / Markdown

Use:

- JSON/JSONL for input samples, predictions, metrics, and machine-readable outputs.
- TOML for `pyproject.toml` and tool metadata only.
- Markdown README for project layout, input/output formats, config fields, and run commands.

Avoid:

- JSON as the main human-written experiment config.
- TOML as train/eval/inference config by default.
- README files that only contain a single run command.

## Read Detailed Rules When Needed

For non-trivial generation or refactoring, read:

- `references/engineering-standard.md`

Use the reference when creating a project, modifying multiple file types, or resolving edge cases around config, paths, logging, CLI overrides, GPU launch, or README structure.
