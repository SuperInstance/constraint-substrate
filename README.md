# constraint-substrate

Five constraint primitives implemented in Rust, C, and Python. Each primitive is independent — use them individually or together.

The five primitives:

| Primitive | What it does |
|-----------|-------------|
| **lattice** | Snap 2D points to Eisenstein A₂ lattice |
| **funnel** | Deadband convergence with exponential decay |
| **holonomy** | Winding number of a cyclic value sequence |
| **rigidity** | Laman's condition check for graph rigidity |
| **consensus** | Metronome consensus rounds with circular mean |

## Lattice Snap

Snap a 2D point to the nearest Eisenstein integer lattice point. The Eisenstein integers Z[ω] (where ω = e^{2πi/3}) form a hexagonal lattice with basis vectors e₁ = (1, 0) and e₂ = (-½, √3/2).

**Python:**
```python
from constraint_substrate import snap

sx, sy, err = snap(0.5, 0.5, group_order=3)
# (0.5, 0.8660254037844386, 0.3660254037844386)
```

**Rust:**
```rust
use constraint_substrate::lattice;

let (sx, sy, err) = lattice::snap(0.5, 0.5, 3);
```

**C:**
```c
#include "constraint_substrate.h"

CsSnapResult r = cs_snap(0.5, 0.5, 3);
// r.a = 0.5, r.b = 0.866..., r.error = 0.366...
```

Algorithm: convert to approximate axial coordinates (b ≈ 2y/√3, a ≈ x + y/√3), then check the 9 candidate lattice points in a 3×3 neighborhood of the floor values.

## Funnel Step

Deadband convergence toward a target with exponentially decaying epsilon. Epsilon shrinks as `epsilon × exp(-decay_rate)` — never reaches zero.

**Python:**
```python
from constraint_substrate import funnel_step

# Outside deadband: pull toward target by one epsilon step
val, eps = funnel_step(current=0.0, target=5.0, epsilon=1.0, decay_rate=0.1)
# val=1.0, eps=0.9048

# Inside deadband: snap toward target with correction
val, eps = funnel_step(current=1.0, target=1.05, epsilon=0.2, decay_rate=0.1)
# val=1.0048, eps=0.1810
```

**Rust:**
```rust
use constraint_substrate::funnel;

let (val, new_eps) = funnel::step(0.0, 5.0, 1.0, 0.1);
// val = 1.0, new_eps = 0.9048
```

**C:**
```c
CsFunnelResult r = cs_funnel_step(0.0, 5.0, 1.0, 0.1);
// r.value = 1.0, r.epsilon = 0.9048
```

Batch variants process N agents at once:

```python
from constraint_substrate import funnel_batch

vals, epsilons = funnel_batch(
    currents=[0.0, 1.0, 2.0],
    targets=[5.0, 1.05, 10.0],
    epsilons=[1.0, 0.2, 0.5],
    decay=0.1,
)
```

## Holonomy / Winding Number

Compute the winding number of a sequence of cyclic values. Each consecutive difference is wrapped to `[-modulus/2, modulus/2]` before summing.

**Python:**
```python
from constraint_substrate import holonomy_winding

# Linear progression: winding = 1.0 (one full wrap)
w = holonomy_winding([1.0, 3.0, 5.0, 7.0, 9.0, 1.0], modulus=10.0)
# 1.0

# No wrapping: winding = 0.3
w = holonomy_winding([1.0, 2.0, 3.0, 4.0], modulus=10.0)
# 0.3
```

**Rust:**
```rust
use constraint_substrate::holonomy;

let w = holonomy::winding(&[1.0, 3.0, 5.0, 7.0, 9.0, 1.0], 10.0);
// 1.0
```

**C:**
```c
double values[] = {1.0, 3.0, 5.0, 7.0, 9.0, 1.0};
double w = cs_holonomy(values, 6, 10.0);
// 1.0
```

## Rigidity (Laman's Condition)

Check if a graph is generically rigid in 2D using Laman's theorem:
1. `|E| ≥ 2|V| - 3`
2. Every subset of k ≥ 2 vertices spans at most `2k - 3` edges

For n ≤ 10: exact enumeration of all subsets. For n > 10: probabilistic sampling (Python) or skipped (Rust/C).

**Python:**
```python
from constraint_substrate import is_laman

# Triangle: minimally rigid
is_laman(3, [(0,1), (1,2), (0,2)])  # True

# Single edge on 3 vertices: not rigid
is_laman(3, [(0,1)])  # False

# K4 (complete graph on 4): violates Laman subgraph condition
is_laman(4, [(0,1),(0,2),(0,3),(1,2),(1,3),(2,3)])  # False
```

**Rust:**
```rust
use constraint_substrate::rigidity;

let rigid = rigidity::is_laman(3, &[(0, 1), (1, 2), (0, 2)]);
// true
```

**C:**
```c
CsEdge edges[] = {{0,1}, {1,2}, {0,2}};
int rigid = cs_is_laman(3, edges, 3);
// 1 (true)
```

## Consensus Rounds

One round of metronome consensus: values within epsilon of the mean snap to the mean; outliers step toward the mean by `epsilon/2`. Supports circular mean for cyclic/phase values.

**Python (linear):**
```python
from constraint_substrate import consensus_round

vals, converged = consensus_round([1.0, 2.0, 3.0], epsilon=0.5)
# vals = [1.25, 2.0, 2.75], converged = False
```

**Python (circular):**
```python
# Values near wrap-around on modulus=1.0: circular mean gives ~0.0, not 0.5
vals, converged = consensus_round([0.1, 0.9], epsilon=0.5, modulus=1.0)
```

**Rust:**
```rust
use constraint_substrate::consensus;

let (vals, converged) = consensus::round(&[1.0, 2.0, 3.0], 0.5, None);
// vals = [1.25, 2.0, 2.75], converged = false

// With circular mean
let (vals, converged) = consensus::round(&[0.1, 0.9], 0.5, Some(1.0));
```

**C:**
```c
double values[] = {1.0, 2.0, 3.0};
CsConsensusResult r = cs_consensus(values, 3, 0.5);
// r.values = [1.25, 2.0, 2.75], r.converged = 0

// With circular mean (modular)
CsConsensusResult r2 = cs_consensus_mod(values, 3, 0.5, 1.0);
cs_consensus_free(r);
cs_consensus_free(r2);  // free allocated arrays
```

## Cross-Language Comparison

| Aspect | Rust | C | Python |
|--------|------|---|--------|
| Location | `rust/src/lib.rs` | `c/src/*.c` | `python/constraint_substrate/*.py` |
| `no_std` | Yes (with `alloc`) | N/A | N/A |
| Dependencies | None | `math.h`, `stdlib.h` | `math` (stdlib) |
| Rigidity n>10 | Skips subgraph check | Skips subgraph check | Random sampling (seed=42) |
| Consensus circular | `Option<f64>` modulus | Separate `cs_consensus_mod` func | `modulus` parameter |
| Batch ops | `snap_batch`, `step_batch` | Single ops only | `snap_batch`, `step_batch` |

## Test Vectors

`tests/vectors.json` contains canonical inputs and expected outputs for all five primitives. Each implementation (Rust, C, Python) runs against the same vectors in CI.

Example vector:
```json
{
  "lattice": {
    "cases": [
      {"input": {"x": 0.5, "y": 0.5, "group_order": 3},
       "output": {"x": 0.5, "y": 0.8660254037844386, "error": 0.3660254037844386}}
    ]
  },
  "holonomy": {
    "cases": [
      {"input": {"values": [1,3,5,7,9,1], "modulus": 10},
       "output": {"winding": 1.0}}
    ]
  }
}
```

## Building

**Rust:**
```bash
cd rust
cargo test
cargo build --release
```

**C:**
```bash
cd c
make clean && make test
```

**Python:**
```bash
cd python
pip install -e .
pytest tests/
```

## Running All Tests

```bash
# From repo root — CI does this via .github/workflows/ci.yml
cd rust && cargo test && cd ../c && make test && cd ../python && pytest tests/
```

## License

MIT
