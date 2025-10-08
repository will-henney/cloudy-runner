# Hydra + Jinja2 template for managing Cloudy runs

> This came from chatpgt 2025-10-07 

Below is a minimal, reproducible project layout and code you can drop into a repo to organize Cloudy runs using **Hydra** (config composition & sweeps) and **Jinja2** (to render Cloudy `.in` files). It mirrors the “noweb” approach by factoring repeated blocks (radiation, physical, saves) into config groups you can mix‑and‑match.

---

## Directory layout

```
cloudy-hydra/
  src/
    run_cloudy.py
  templates/
    cloudy.in.j2
  configs/
    config.yaml
    run/
      base.yaml
      fast.yaml
      slow.yaml
    radiation/
      powr_ob_smc.yaml
      blackbody.yaml
    physical/
      const_n10.yaml
      constP_n10.yaml
      constP_n30.yaml
    saves/
      lines_and_bands.yaml
      minimal.yaml
```

> Tip: keep your SED files under a known path and pass that via config (e.g. `sed_path`).

---

## `templates/cloudy.in.j2`

A generic Cloudy input template. Jinja2 fills in blocks from Hydra configs.

```jinja2
{# Title #}
{{ title }}

{# Physical / density / B-field / geometry block #}
{% for line in physical_block %}
{{ line }}
{% endfor %}

{# Radiation field block #}
{% for line in radiation_block %}
{{ line }}
{% endfor %}

{# Stopping criteria, time-stepping, etc., if any #}
{% for line in runtime_block %}
{{ line }}
{% endfor %}

{# Save lines / continua / grids etc. #}
{% for line in saves_block %}
{{ line }}
{% endfor %}
```

---

## `configs/config.yaml` (root)

```yaml
defaults:
  - run: base
  - radiation: powr_ob_smc
  - physical: const_n10
  - saves: lines_and_bands

# Cloudy binary and I/O
cloudy:
  binary: cloudy          # your executable name in PATH (or absolute path)
  workdir: ${hydra:run.dir}
  input_basename: w3      # basename for files, we’ll add suffixes per run

# Shared runtime options (can be extended per-run)
runtime_block: []

title: "Walborn 3 in SMC-N66 / NGC 346"

# Optional extra values used by specific blocks
sed_path: "/path/to/your/seds"  # used by radiation configs
```

---

## `configs/radiation/powr_ob_smc.yaml`

```yaml
# POWR O-star SED (SMC metallicity) example
# Adapt this to point at your converted Cloudy SED files
radiation_block:
  - table SED file "${sed_path}/smc-ob-i_50-42.sed"
  - phi(H) 49  # example ionizing photon flux, replace with your value
```

## `configs/radiation/blackbody.yaml`

```yaml
radiation_block:
  - blackbody 50000 K
  - luminosity total 38  # log L/Lsun or as needed
```

---

## `configs/physical/const_n10.yaml`

```yaml
physical_block:
  - hden 1.00                # 10 cm^-3
  - constant density
  - magnetic field -5.3 gauss
```

## `configs/physical/constP_n10.yaml`

```yaml
physical_block:
  - hden 1.00                # initial guess
  - constant pressure
  - magnetic field -5.3 gauss
```

## `configs/physical/constP_n30.yaml`

```yaml
physical_block:
  - hden 1.48                # ~30 cm^-3 (log10 30 ≈ 1.48)
  - constant pressure
  - magnetic field -5.3 gauss
```

---

## `configs/saves/lines_and_bands.yaml`

```yaml
# Extend with whatever you routinely save from Cloudy
saves_block:
  - save lines emissivity last "lines_emis.out"
  - save continuum units Angstrom last "cont_last.out"
  - save continuum bands last "cont_bands.out"
```

## `configs/saves/minimal.yaml`

```yaml
saves_block:
  - save last "last.out"
```

---

## `configs/run/base.yaml`

```yaml
# Per-run tweaks and suffix for naming
run:
  suffix: "n10"

# You can also extend runtime controls here (e.g., stopping criteria)
runtime_block:
  - stop temperature 100 K
```

## `configs/run/fast.yaml`

```yaml
defaults:
  - override /run: base

run:
  suffix: "fast"

runtime_block:
  - set save prefix "FAST_"
  - iterate to convergence
```

## `configs/run/slow.yaml`

```yaml
defaults:
  - override /run: base

run:
  suffix: "slow"

runtime_block:
  - set save prefix "SLOW_"
  - iterate to convergence
  - punches "additional controls here"
```

---

## `src/run_cloudy.py`

```python
import subprocess
from pathlib import Path
from typing import List

import hydra
from omegaconf import DictConfig, OmegaConf
from jinja2 import Environment, FileSystemLoader, StrictUndefined


def render_template(template_dir: Path, context: dict) -> str:
    env = Environment(
        loader=FileSystemLoader(str(template_dir)),
        undefined=StrictUndefined,
        trim_blocks=True,
        lstrip_blocks=True,
    )
    tmpl = env.get_template("cloudy.in.j2")
    return tmpl.render(**context)


@hydra.main(version_base=None, config_path="../configs", config_name="config")
def main(cfg: DictConfig) -> None:
    run_dir = Path(cfg.cloudy.workdir)
    run_dir.mkdir(parents=True, exist_ok=True)

    # Compose final blocks (root runtime + run-specific runtime)
    runtime_block: List[str] = []
    runtime_block.extend(cfg.get("runtime_block", []))
    runtime_block.extend(cfg.get("run", {}).get("runtime_block", []))

    # Build the Jinja context from composed Hydra config
    context = {
        "title": f"{cfg.title}, {cfg.run.suffix}",
        "physical_block": cfg.physical.physical_block,
        "radiation_block": cfg.radiation.radiation_block,
        "runtime_block": runtime_block,
        "saves_block": cfg.saves.saves_block,
    }

    # Render Cloudy input text
    template_dir = Path(__file__).resolve().parent.parent / "templates"
    input_text = render_template(template_dir, context)

    # Write input file (name drives Cloudy’s output names when using stdin redirection)
    input_name = f"{cfg.cloudy.input_basename}-{cfg.run.suffix}.in"
    input_path = run_dir / input_name
    input_path.write_text(input_text)

    # Run Cloudy by feeding the .in file to stdin so filenames are preserved
    # Typical Cloudy usage: `cloudy < input.in`
    proc = subprocess.run(
        [cfg.cloudy.binary],
        cwd=run_dir,
        stdin=open(input_path, "rb"),
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE,
        check=False,
    )

    # Save stdout/stderr for debugging
    (run_dir / f"{cfg.cloudy.input_basename}-{cfg.run.suffix}.stdout").write_bytes(proc.stdout)
    (run_dir / f"{cfg.cloudy.input_basename}-{cfg.run.suffix}.stderr").write_bytes(proc.stderr)

    # Also snapshot the *resolved* config for reproducibility
    (run_dir / "resolved-config.yaml").write_text(OmegaConf.to_yaml(cfg))

    if proc.returncode != 0:
        raise SystemExit(f"Cloudy exited with code {proc.returncode}")


if __name__ == "__main__":
    main()
```

---

## How to run

From the project root:

```bash
# Single run with defaults
python src/run_cloudy.py

# Switch physical model (constant pressure @ n~10)
python src/run_cloudy.py physical=constP_n10 run.suffix=constP-n10

# Switch to n~30 constant pressure and a blackbody source
python src/run_cloudy.py physical=constP_n30 radiation=blackbody run.suffix=constP-n30-bb

# Parameter sweep via multirun (creates one output dir per combo)
python src/run_cloudy.py --multirun \
  physical=const_n10,constP_n10,constP_n30 \
  radiation=powr_ob_smc,blackbody \
  run=fast,slow
```

Hydra will create unique run directories (e.g. `./multirun/…`) and snapshot the full resolved config for each run, ensuring full reproducibility.

---

## Notes on mapping your Org “noweb” blocks

- Your `<<cloudy-w3-radiation>>`, `<<cloudy-w3-common-physical>>`, and `<<cloudy-saves>>` map directly to the **config groups** `radiation/`, `physical/`, and `saves/`.
- Each YAML file in those groups corresponds to a reusable snippet (a list of Cloudy commands) that the Jinja template injects in the correct position.
- You can create multiple variants (e.g., different B‑fields, densities, and SEDs) and compose them per run without editing code.
- If you later want stricter typing and defaults, migrate these lists to **structured configs** (dataclasses) and render with the same template.

---

## Optional extensions

- **Cluster / Slurm**: swap Hydra’s default local launcher for a Slurm launcher (e.g., `hydra-submitit-launcher`) to fan out sweeps on HPC. Each job will run `cloudy` in its own working dir.
- **Result harvesting**: after `cloudy` finishes, parse outputs and write a small CSV/Parquet summary with the run directory and key diagnostics. Expose output columns via config so you can customize per project.
- **SED registries**: create a `configs/seds/*.yaml` group that maps symbolic SED names to absolute file paths; your `radiation` group can then reference `${seds.powr_ob_i_50_42}` to avoid hard‑coding paths.
- **Packaging**: add a `pyproject.toml` and use `uv` to manage the environment; declare deps (`hydra-core`, `jinja2`).

---

This gives you the same modularity you had with Org/noweb, but in a toolchain that’s easy for collaborators (no Emacs required) and plays nicely with parameter sweeps, cluster runs, and reproducibility snapshots.

