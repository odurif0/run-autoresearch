# autoresearch-optimize

A Forge skill for autonomous parameter/code optimization through systematic trial-and-error with automatic KEEP/DISCARD decisions.

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

Your project needs:

1. **Benchmark script** (`benchmark/benchmark.{jl,py,R,rs,cpp,...}`) that outputs:
   ```
   METRIC_JSON
   {"primary":-2825.5,"time_s":11.84,"success":true,...}
   ```

2. **Evaluate script** (`benchmark/evaluate.sh`) that compares against baseline:
   ```bash
   ./benchmark/evaluate.sh benchmark/baseline.json /tmp/result.txt
   # Exit 0 = KEEP, Exit 1 = DISCARD, Exit 2 = ERROR
   ```

3. **Baseline file** (`benchmark/baseline.json`) with current best metrics.

## Installation

### As a Forge skill (recommended)

```bash
mkdir -p ~/.forge/skills
git clone https://github.com/odurif0/autoresearch-optimize ~/.forge/skills/autoresearch-optimize
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
autoresearch-optimize/
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
