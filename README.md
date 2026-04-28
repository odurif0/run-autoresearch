# run-autoresearch

A **Forge skill** (not a standalone agent) for autonomous parameter/code optimization through systematic trial-and-error with automatic KEEP/DISCARD decisions. The executing agent remains **Forge** -- this skill provides the methodology, scripts, and reference material that Forge follows during optimization sessions.

Designed as a generic, language-agnostic optimization loop: **propose change -> benchmark -> evaluate -> KEEP or DISCARD -> repeat**.

Inspired by [karpathy/autoresearch](https://github.com/karpathy/autoresearch) and refined with strategies from the autoresearch community (failure-driven planning, variance-aware acceptance, structured experiment logging).

## How it works

```
┌─────────────────────────────────────────────┐
│  Phase 0: Setup & Read project context      │
│  Auto-create baseline, evaluate.sh, TSV     │
│  Read program.md, source, history           │
├─────────────────────────────────────────────┤
│  Phase 1: Plan wave (failure-driven)        │
│  6-10 experiments, prioritized by leverage  │
├─────────────────────────────────────────────┤
│  Phase 2: Execute experiments               │
│  ┌──────────────────┐   ┌────────────────┐  │
│  │ Apply 1 change   │──>│ Verify compile │  │
│  └──────────────────┘   └───────┬────────┘  │
│  ┌──────────────────┐   ┌───────▼────────┐  │
│  │ Revert (DISCARD) │<──│ Run benchmark  │  │
│  └──────────────────┘   └───────┬────────┘  │
│  ┌──────────────────┐   ┌───────▼────────┐  │
│  │ Commit (KEEP)    │<──│ Evaluate       │  │
│  └──────────────────┘   └────────────────┘  │
├─────────────────────────────────────────────┤
│  Phase 3: Wave summary + plateau detection  │
│  -> Plan next wave or declare optimum       │
└─────────────────────────────────────────────┘
```

## Key features

- **Failure-driven experiment planning** -- analyzes previous wave results to prioritize the most promising changes, rather than blind grid search
- **Runtime profile awareness** -- adapts measurement strategy per language (warmup for JIT, multi-run for stochastic, single-run for compiled)
- **Variance-aware acceptance** -- confirmation runs for marginal improvements, prevents false KEEP from noise
- **Structured experiment log** -- append-only TSV for reliable session resume and pattern analysis
- **Plateau recovery** -- 5 escalation strategies before declaring a true optimum
- **15 anti-patterns** -- catalog of changes that typically fail, to avoid wasting benchmark runs
- **18 exploration axes** -- generic dimensions organized by expected leverage (algorithmic > constraints > hyperparameters > micro-optimizations)

## Prerequisites

No manual setup required — Phase 0 auto-configures everything on first run:

**Benchmark script** (`benchmark/benchmark.{jl,py,R,rs,cpp,...}`) — either user-provided or LLM-generated during Phase 0. Must output:

```
METRIC_JSON
{"primary":-2825.5,"time_s":11.84,"success":true,...}
```

The JSON must contain at minimum: `success` (bool) and the primary metric.

**If the benchmark is missing, Phase 0 generates it** by reading your project source code — just validate the result before experiments begin.

Phase 0 auto-creates everything else:

| File | Action |
|------|--------|
| `benchmark/` dir | Created if missing |
| `benchmark/evaluate.sh` | Generated with default KEEP/DISCARD logic |
| `benchmark/baseline.json` | Created by running the benchmark |
| `benchmark/experiments.tsv` | Created on first experiment log |

## Writing a benchmark script

> **The benchmark is the foundation of the entire optimization loop.** Every KEEP/DISCARD decision depends on it. If the benchmark doesn't exercise the right pipeline, uses the wrong metric, or is non-deterministic, the optimizer will converge to garbage. Whether you write it yourself or let the LLM generate it, validate it carefully.

Here's what a good benchmark looks like:

### Required format

The script must output **exactly** this pattern to stdout:

```
any log lines, warnings, progress...
METRIC_JSON
{"primary":-2825.5,"time_s":11.84,"success":true,"r2":0.9994,"n_peaks":12}
```

- `METRIC_JSON` on its own line signals that the next line is the JSON payload
- The JSON line must be a single line (no pretty-printing)
- Exit code 0 on success

### Mandatory JSON fields

| Field | Type | Description |
|-------|------|-------------|
| `success` | bool | `true` if the benchmark completed successfully |
| `time_s` | float | Wall-clock time in seconds for the actual computation |

Plus at least **one primary metric** field — this is the value the optimizer tries to improve. Name it something clear:

- `bic`, `accuracy`, `primary`, `loss`, `r2`, `latency_ms`, `throughput`, `f1_score`, `mae`

### Optional JSON fields

Any additional metrics you want tracked in the TSV log — `r2`, `n_peaks`, `n_models`, `memory_mb`, etc.

### Design principles

1. **Deterministic**: Use a fixed random seed. Non-deterministic runs produce unreliable comparisons.
2. **Self-contained**: Hardcode the test data path. Don't depend on environment variables or user input.
3. **Reasonable duration**: 5-30 seconds is ideal. Too fast (< 1s) = noisy measurements. Too slow (> 2 min) = too few experiments per session.
4. **Warmup-aware for JIT languages**: If using Julia/Java/C#, do a warmup run first and measure the second run. Or let `run_bench.sh` handle it (it auto-detects hybrid profiles and runs a warmup pass).
5. **Capture failures**: If the computation fails mid-run, catch the exception and write `{"success":false, ...}` — don't crash. This lets the optimizer DISCARD gracefully instead of aborting the wave.
6. **Sweep the full pipeline**: If your code finds multiple solutions and picks the best one, include the full sweep in the benchmark. Don't benchmark a single fixed-config run — test what the optimizer actually does.

### Minimal examples

**Julia:**
```julia
#!/usr/bin/env julia
using MyPackage
RNG = Random.MersenneTwister(42)
data = MyPackage.load_data("test/data.txt")
start = time()
result = MyPackage.run_optimizer(data; seed=RNG)
elapsed = time() - start
@info "Done" elapsed_s=elapsed bic=result.bic n=result.n_components
println("METRIC_JSON")
println("""{"bic":$(result.bic),"time_s":$(round(elapsed;digits=3)),"success":true,"r2":$(result.r2)}""")
```

**Python:**
```python
#!/usr/bin/env python3
import time, json, numpy as np
np.random.seed(42)
from mypackage import load_data, run_optimizer

data = load_data("test/data.txt")
start = time.time()
result = run_optimizer(data)
elapsed = time.time() - start

print("METRIC_JSON")
print(json.dumps({
    "bic": result.bic, "time_s": round(elapsed, 3),
    "success": True, "r2": result.r2
}))
```

### Bad benchmark checklist

These flaws will poison the optimization loop. Before trusting results, verify none of these apply:

| Red flag | Why it breaks the loop |
|----------|------------------------|
| No fixed seed | Primary metric varies between runs -> KEEP/DISCARD becomes random |
| Runs unit tests instead of the optimizer | Measures test pass rate, not optimization quality -> meaningless delta |
| Times the wrong code path | Benchmark says "improved 20%" but the optimizer is unchanged |
| Primary metric = `success` boolean | No gradation — can't detect 1% improvements |
| JSON line is pretty-printed or wrapped | `METRIC_JSON` extraction fails -> crash |
| Uses environment variables for data path | Unreliable across sessions -> baseline can't be reproduced |
| Runs once, takes a median internally | Prevents external warmup/multi-run protocol |
| Exits non-zero on optimization failure | Should output `{"success":false,...}` instead -> crash vs DISCARD |
| Hardcodes parameter values that will be changed | Parameter is hardcoded in benchmark -> changing source has no effect |

### Sanity check

Before starting any experiment wave, run this manual check:

```bash
# Run benchmark twice with no code changes
bash scripts/run_bench.sh sanity-1 benchmark/baseline.json
bash scripts/run_bench.sh sanity-2 benchmark/baseline.json
```

Both must produce the **same primary metric within 0.5%**. If they don't, the benchmark is non-deterministic and needs fixing.

## Installation

### As a Forge skill (recommended)

```bash
mkdir -p ~/.forge/skills
git clone https://github.com/odurif0/autoresearch-optimize ~/.forge/skills/run-autoresearch
```

The skill will be automatically available in Forge sessions. Invoke it by saying:

> "Run autoresearch optimization on this project"
> "Resume the optimization session"
> "Plan a new wave of experiments"

### Standalone

The `scripts/run_bench.sh` helper can be used independently:

```bash
# Initialize baseline
bash scripts/run_bench.sh init-baseline benchmark/baseline.json

# Run an experiment
bash scripts/run_bench.sh my-experiment benchmark/baseline.json
```

## Runtime profiles

| Profile | Languages | Warmup | Variance |
|---------|-----------|--------|----------|
| `compiled` | C, C++, Rust, Go, Fortran | 0 | ~1-2% |
| `hybrid` | Julia, Java, C# | 1 run | ~5-10% |
| `interpreted` | Python, R, Ruby, MATLAB | 0 | ~3-8% |
| `stochastic` | (any, unfixed seed) | per language | High |

Auto-detected from benchmark script extension. Override with:
```bash
export AUTORESEARCH_PROFILE=compiled
```

## Decision logic

Default KEEP/DISCARD rule (adjust via `evaluate.sh`):

```
KEEP if:
  - Primary metric improved by > 2%
  OR
  - Primary metric stable within +/-2% AND time improved by > 15%

DISCARD otherwise (crash = auto-DISCARD)
```

Marginal improvements (2-3%) trigger a confirmation run before KEEPing.

## File structure

```
run-autoresearch/
├── SKILL.md                  # Complete workflow specification
├── scripts/
│   └── run_bench.sh          # Benchmark runner with profile detection
└── references/
    ├── anti-patterns.md      # 15 generic anti-patterns to avoid
    └── axes.md               # 18 exploration axes by leverage level
```

## Acknowledgments

Key ideas borrowed from the autoresearch community:

- **[karpathy/autoresearch](https://github.com/karpathy/autoresearch)** -- the original concept of autonomous code optimization
- **[brainforge-autoresearch](https://github.com/zning1994/brainforge-autoresearch)** -- failure-driven mutation, structured changelog
- **[awesome-autoresearch](https://github.com/yibie/awesome-autoresearch)** -- community catalog of tools and patterns
- **[helix](https://github.com/VectorInstitute/helix)** -- normalized experiment format, append-only TSV
- **[Artificial General Research](https://github.com/JoaquinMulet/Artificial-General-Research)** -- variance-aware acceptance

## License

MIT

---

*Created with Deepseek V4 Pro*
