# Engineering Standard Reference

This reference expands the personal engineering style. Use it when generating or modifying multi-file Python/YAML/Bash projects.

## 1. Project Layout

Default layout:

```text
project/
├── README.md
├── requirements.txt
├── configs/
│   └── inference.yaml
├── scripts/
│   └── run_inference.sh
├── data/
├── outputs/
└── src/
    └── package_name/
        ├── __init__.py
        ├── cli.py
        ├── config.py
        ├── io.py
        └── model.py
```

Use this layout for long-lived projects. For tiny one-off scripts, fewer files are acceptable only when the user asks for something small.

Responsibilities:

- `cli.py`: command-line parsing and subcommand dispatch.
- `config.py`: YAML loading, config override helpers, path resolution.
- `io.py`: JSONL and file I/O helpers.
- `model.py`: model/policy/environment classes.
- `scripts/`: Bash launchers only; no business logic.
- `configs/`: YAML configs.
- `outputs/`: predictions, logs, metrics.

Avoid broad `utils.py` files unless the helper is genuinely generic and small.

## 2. Python Rules

### Imports

Group imports as standard library, third-party, local package.

```python
import argparse
import json
import logging
from pathlib import Path
from typing import Iterable, Iterator

import yaml
from tqdm import tqdm

from package_name.config import load_yaml, resolve_config_path
from package_name.model import GraspPolicy
```

Avoid wildcard imports.

### Naming

Use full, clear names:

```python
config_path
checkpoint_path
input_path
prediction_path
batch_size
score_threshold
```

Avoid:

```python
cfgp
ckpt
inp
pred
bs
thr
```

### CLI

Default to subcommands:

```python
parser = argparse.ArgumentParser(description="Robot inference CLI.")
subparsers = parser.add_subparsers(dest="command", required=True)

infer_parser = subparsers.add_parser("infer")
infer_parser.add_argument("--config", required=True)
infer_parser.add_argument("--device", default=None)
infer_parser.add_argument("--batch-size", type=int, default=None)
```

Priority:

```text
CLI explicit arguments > YAML config > code fallback defaults
```

### Config Loading

Prefer dict-based YAML config with centralized extraction.

```python
def load_yaml(path: Path) -> dict:
    with path.open("r", encoding="utf-8") as f:
        data = yaml.safe_load(f)
    if not isinstance(data, dict):
        raise ValueError(f"Config must be a mapping: {path}")
    return data
```

Do not default to dataclass or pydantic config objects. Use them only when the user asks for stronger validation or the project is clearly large enough to benefit.

### Paths

Resolve YAML paths relative to the config file:

```python
def resolve_config_path(config_path: Path, path_value: str) -> Path:
    path = Path(path_value)
    return path if path.is_absolute() else config_path.parent / path
```

Do not rely on the current working directory.

### Functions vs Classes

Prefer linear command handlers and helper functions.

Use classes for naturally stateful objects:

```python
class GraspPolicy:
    def __init__(self, checkpoint_path: Path, device: str) -> None:
        self.checkpoint_path = checkpoint_path
        self.device = device

    def predict(self, sample: dict, score_threshold: float) -> dict:
        ...
```

Avoid default `Pipeline`, `Manager`, `Factory`, `Registry`, `Dataset`, or `Writer` classes for simple scripts.

### Type Hints

Add simple type hints to key boundaries:

```python
def read_jsonl(path: Path) -> Iterator[dict]:
    ...

def write_jsonl(path: Path, records: Iterable[dict]) -> int:
    ...
```

Avoid over-specific generics unless needed.

### Logging

Use:

```python
logger = logging.getLogger(__name__)
logging.basicConfig(level=logging.INFO, format="%(levelname)s %(message)s")
```

Log at least:

- config path
- checkpoint path
- input path
- output path
- device
- processed count

Avoid JSON logging by default.

### Errors

Check critical files:

```python
if not input_path.exists():
    raise FileNotFoundError(f"Input file not found: {input_path}")

if not checkpoint_path.exists():
    raise FileNotFoundError(f"Checkpoint not found: {checkpoint_path}")
```

For JSONL:

```python
try:
    yield json.loads(line)
except json.JSONDecodeError as exc:
    raise ValueError(f"Invalid JSONL at {path}:{line_number}") from exc
```

Do not swallow exceptions with broad `except Exception`.

### Inference Loop

Use `tqdm` for long tasks:

```python
for sample in tqdm(read_jsonl(input_path), desc="infer"):
    prediction = model.predict(sample, score_threshold)
```

Use batch inference only when it changes real model behavior or performance.

## 3. YAML Rules

Default structure:

```yaml
# Relative paths are resolved from this config file.

model:
  name: grasp_policy
  checkpoint_path: ../checkpoints/grasp_policy.pt

dataset:
  input_path: ../data/samples.jsonl
  limit: null

train:
  enabled: false

eval:
  enabled: false

inference:
  enabled: true
  batch_size: 8
  score_threshold: 0.55
  prediction_path: ../outputs/predictions.jsonl

runtime:
  device: cuda:0
  dtype: float16
  seed: 42
```

Rules:

- Use `snake_case`.
- Prefer clear field names.
- Use `true`/`false`, not `True`/`False`.
- Use `null`, not empty strings.
- Keep comments sparse.
- Put device, dtype, and seed under `runtime`.
- Put checkpoint under `model`.
- Put output path under the command-specific group, such as `inference.prediction_path`.

## 4. Bash Rules

Use strict Bash launchers:

```bash
#!/usr/bin/env bash
set -euo pipefail

PROJECT_ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
CONFIG_PATH="${CONFIG_PATH:-${PROJECT_ROOT}/configs/inference.yaml}"
CUDA_VISIBLE_DEVICES="${CUDA_VISIBLE_DEVICES:-0}"
DEVICE="${DEVICE:-cuda:0}"
BATCH_SIZE="${BATCH_SIZE:-}"
LOG_DIR="${PROJECT_ROOT}/outputs/logs"

export CUDA_VISIBLE_DEVICES
export PYTHONPATH="${PROJECT_ROOT}/src:${PYTHONPATH:-}"

mkdir -p "${LOG_DIR}"

COMMAND=(
  python -m package_name.cli infer
  --config "${CONFIG_PATH}"
  --device "${DEVICE}"
)

if [[ -n "${BATCH_SIZE}" ]]; then
  COMMAND+=(--batch-size "${BATCH_SIZE}")
fi

"${COMMAND[@]}" 2>&1 | tee "${LOG_DIR}/inference.log"
```

Rules:

- Quote every variable expansion.
- Keep long commands one argument per line.
- Let environment variables override Bash variables, then pass explicit CLI args.
- Do not make Python silently read environment variables by default.
- Use `tee` for logs.
- Use `PROJECT_ROOT` for script-level paths.

## 5. Data and Config File Types

Use YAML for human-written experiment config.

Use JSON/JSONL for:

- input samples
- predictions
- metrics
- machine-readable outputs

Use TOML for:

- `pyproject.toml`
- tool config
- project metadata

Use `requirements.txt` by default for dependencies:

```text
pyyaml>=6.0
tqdm>=4.66
```

Do not default to Makefile, Dockerfile, or `.env`.

## 6. README Rules

README should include:

- Project layout.
- Setup command.
- Run command.
- Input data format.
- Output data format.
- Key config fields.

Minimum useful outline:

```markdown
# Project Name

## Layout

## Setup

## Run

## Input

## Output

## Config
```

## 7. Train / Eval / Inference Boundaries

Separate commands:

```bash
python -m package_name.cli train --config configs/train.yaml
python -m package_name.cli eval --config configs/eval.yaml
python -m package_name.cli infer --config configs/inference.yaml
```

YAML may include `train`, `eval`, and `inference`, but each command should read only relevant sections.

Avoid mixing training optimizer settings into inference-only code.

