---
name: qbit-viz
description: Use when the user asks about quantum gates, Bloch sphere visualization, qubit state evolution, or Qiskit/Aer simulation. Triggers on "quantum gate", "Bloch sphere", "qubit visualization", "state evolution", "Qiskit", "Aer simulator", "quantum circuit", "布球", "量子態演化".
---

# Quantum Bit Visualization (Qiskit + Aer)

2/3 qubit logic gates, Bloch sphere rendering, state evolution, and interactive demos using Qiskit + Aer.

## Core Invariant

**Every visual element must be derived from an actual quantum state computed by Qiskit (or an Aer measurement result) — never illustrative art.** All later rules follow from this: Qiskit/Aer owns the math, `matplotlib` owns 2D charts, `pyvista` + `panel` own 3D (including Bloch spheres), and **both live in one app** — `qbit-gate-viz/app.py` (Streamlit: 2D + integrated 3D). No separate standalone 3D demo.

## When to Use

- Quantum gate operations (X, Y, Z, H, S, T, CNOT, CZ, SWAP, Toffoli)
- Bloch sphere visualization (single/multi-qubit)
- Quantum state evolution and entanglement
- Qiskit circuit simulation (Aer backend)
- Interactive quantum demos (Streamlit)

## Core Concepts

### Bloch Sphere Representation
A single qubit pure state: $|\psi\rangle = \cos(\theta/2)|0\rangle + e^{i\phi}\sin(\theta/2)|1\rangle$

Coordinates:
- $x = \sin\theta\cos\phi$
- $y = \sin\theta\sin\phi$
- $z = \cos\theta$

### Tensor Product Structure (Multi-qubit)
$$\mathcal{H}_{total} = \mathcal{H}_{n-1} \otimes \dots \otimes \mathcal{H}_{target} \otimes \dots \otimes \mathcal{H}_0$$

Qiskit uses Little-endian: Index 0 is rightmost (LSB). Operator construction:
```python
prefix = "I" * (n_qubits - 1 - qubit_idx)
suffix = "I" * qubit_idx
operator = prefix + gate + suffix  # e.g., "IXI" for middle qubit
```

### Entanglement Effect
When qubits are entangled, individual Bloch vectors shrink or vanish — expected, not a bug. Visualize the 3D state in the integrated 3D section of `qbit-gate-viz/app.py` (PyVista + Panel); do **not** use matplotlib-backed Qiskit renderers.

## Logic Gates

### Single-Qubit Gates
| Gate | Matrix | Bloch Effect | Qiskit |
|------|--------|-------------|--------|
| I | [[1,0],[0,1]] | Identity | `qc.i(q)` |
| X | [[0,1],[1,0]] | 180° rotation around X | `qc.x(q)` |
| Y | [[0,-i],[i,0]] | 180° rotation around Y | `qc.y(q)` |
| Z | [[1,0],[0,-1]] | 180° rotation around Z | `qc.z(q)` |
| H | (1/√2)[[1,1],[1,-1]] | Creates superposition | `qc.h(q)` |
| S | [[1,0],[0,i]] | 90° rotation around Z | `qc.s(q)` |
| T | [[1,0],[0,e^(iπ/4)]] | 45° rotation around Z | `qc.t(q)` |

### Multi-Qubit Gates
| Gate | Effect | Qiskit |
|------|--------|--------|
| CNOT | Flips target if control is 1 | `qc.cx(c, t)` |
| CZ | Phase flip if both are 1 | `qc.cz(c, t)` |
| SWAP | Swaps two qubits | `qc.swap(a, b)` |
| Toffoli (CCNOT) | Flips target if both controls are 1 | `qc.ccx(c1, c2, t)` |
| Fredkin (CSWAP) | Swaps two targets if control is 1 | `qc.cswap(c, t1, t2)` |

> **Selector rule:** never let control/target share a bit index (`CircuitError: duplicate bit arguments`).
> `qbit-viz` owns rendering; follow **`qbit-gate`** for dynamic gate families and mutual-exclusive
> control/target selectors (option filtering at selection time, not post-hoc `st.info` fixes).

## Visualization Methods

### Statevector (math only)
```python
from qiskit.quantum_info import Statevector

sv = Statevector(qc)
probs = sv.probabilities_dict()   # numbers only, no rendering
```

> `plot_bloch_multivector`, `plot_state_qsphere`, `plot_state_city` are **matplotlib-backed 3D** renderers. Do not use them in new demos — use the integrated 3D section of `qbit-gate-viz/app.py` (PyVista + Panel).

### PyVista 3D Rendering (proven implementation)
Proven in `5布球態演化維問.py`: off-screen `Plotter`, sphere `opacity=0.15`,
state vector as `PolyData` line + moving tip sphere, per-frame `screenshot()` streamed to UI.

```python
import os
os.environ["PYVISTA_OFF_SCREEN"] = "True"   # BEFORE importing pyvista
import pyvista as pv

plotter = pv.Plotter(off_screen=True, window_size=(480, 480))
plotter.add_mesh(pv.Sphere(radius=1.0, theta_resolution=32, phi_resolution=24),
                 color="gray", opacity=0.15, show_edges=True)
line = pv.PolyData([[0, 0, 0], [1, 0, 0]])
line.lines = [2, 0, 1]
plotter.add_mesh(line, color="#00ffff", line_width=4)
tip = pv.Sphere(radius=0.06, center=(1, 0, 0))
plotter.add_mesh(tip, color="#ffcc00")
# per frame: line.points[1] = [bx,by,bz]; tip.center = [bx,by,bz];
#           plotter.render(); frame = plotter.screenshot()
plotter.close()
```

Panel integration: `pn.extension("vtk")` then `pn.pane.VTK(plotter.ren_win, width=650, height=650)`
(the pane takes the **render window**, not the Plotter). Close the previous Plotter before rebuilding.

### Aer Simulator
```python
from qiskit_aer import AerSimulator
sim = AerSimulator()
result = sim.run(qc, shots=1024).result()
counts = result.get_counts()   # keys are already bitstrings, e.g. "00"
```

> For a **Streamlit** app, do not create `AerSimulator()` on every rerun — it segfaults.
> Follow `qbit-gate` for `@st.cache_resource`, single-thread options, and mutual-exclusive
> control/target selectors. `qbit-viz` owns rendering; `qbit-gate` owns circuit construction.

## Interactive Demo (Streamlit)

```python
import streamlit as st
from qiskit import QuantumCircuit
from qiskit.quantum_info import Statevector
st.set_page_config(page_title="Qubit Lab", layout="wide")
num_qubits = st.sidebar.radio("Qubits", [1, 2, 3], horizontal=True)
# Gate selection / mutual exclusion: see qbit-gate (option filtering, not st.info)
# Circuit execution here; 3D Bloch lives in the integrated section below (PyVista + Panel)
```

## Code Templates

### 2-Qubit Bell State
```python
from qiskit import QuantumCircuit
from qiskit.quantum_info import Statevector
qc = QuantumCircuit(2)
qc.h(0)
qc.cx(0, 1)
sv = Statevector(qc)
print(sv.probabilities_dict())   # {'00': ~0.5, '11': ~0.5} — entangled
# 3D view: integrated 3D section of qbit-gate-viz/app.py (PyVista + Panel)
```

### 3-Qubit GHZ State
```python
qc = QuantumCircuit(3)
qc.h(0)
qc.cx(0, 1)
qc.cx(0, 2)
sv = Statevector(qc)
print(sv.probabilities_dict())   # {'000': ~0.5, '111': ~0.5} — GHZ
# 3D view: integrated 3D section of qbit-gate-viz/app.py (PyVista + Panel)
```

## Streamlit Rules (from viz-dboard)

When building Streamlit-based quantum visualization demos, apply these rules:

- **Single entry point**: `app.py` with `st.set_page_config` called once + `src/views/` modules + `requirements.txt`
- **No `pages/` directory or `st.navigation`**: causes StreamlitAPIException on Streamlit Cloud
- **Sidebar radio pattern**: theme blocks provide `<name>_sidebar()` with `horizontal=True`, app.py calls conditionally
- **`__init__.py` sync**: keep `__init__.py` and `app.py` content in sync (supports `streamlit run __init__.py`)
- **Relative paths**: use relative paths for assets (avoid cloud path errors)
- **Local testing**: `streamlit run app.py`
- **Deployment target**: Streamlit Cloud only (interactive, needs GitHub login)
- **Widget choice**: Use `st.radio` instead of `st.selectbox`
- **Layout**: Use `st.columns([3, 7])` for left-right layout
- **Container width**: `use_container_width=True` → `width='stretch'`. `False` → `width='content'`
- **Context manager**: Use `with` identifier, not `expand`
- **Charting library**: 2D → **`matplotlib` only** (no `seaborn`, no `PIL`, no `plotly`). 3D → **`pyvista` + `panel`** unconditionally, including Bloch spheres — no matplotlib-3D exception. See the integrated 3D section of `qbit-gate-viz/app.py` and `5布球態演化維問.py`.
- **Implementation prerequisite (owned by `viz-dboard`)**: before writing or changing any demo `app.py`, read `~/荐家/布球/INDEX.md` + the matching `*_distilled.md` from `file-distill` **before** raw `.py` scripts or sibling demos. Do not duplicate the rule here — follow `viz-dboard` → Demo Implementation Prerequisite — Distilled-First.

## Demo Tools

| Tool | Location | Startup | Description |
|------|----------|---------|-------------|
| qbit-gate-viz | `~/荐家/布球/qbit-gate-viz/app.py` | `streamlit run app.py` | **One integrated app**: Qiskit + Aer 1/2/3 qubit gates, mutual-exclusive control/target, 2D Bloch projection (matplotlib), **plus 3D Bloch evolution** (Rx/Ry/Rz, trail, α/β + θ/φ readout) via PyVista → Panel VTK pane → `st.iframe` |

> `~/荐家/布球/bloch-evolution-viz/app.py` is a **retired standalone demo** — its logic was folded into `qbit-gate-viz/app.py`. Do not list or copy it.
> `quant-dboard` is a **quant finance** dashboard (equity / drawdown / Kelly), NOT a Qiskit demo. Do not list it here.

## Guardrails
1. Entanglement shrinks Bloch vectors — this is expected, not a bug.
2. For entangled states, use the integrated 3D section of `qbit-gate-viz/app.py` (PyVista + Panel); never `plot_state_qsphere` (matplotlib-backed).
3. Always normalize state vectors before visualization.
4. Aer simulator uses `shots` parameter for measurement statistics.

## References
- `qbit-gate` — circuit construction, mutual-exclusive indices, Aer rerun safety (child capability)
- Qiskit documentation: https://docs.quantum.ibm.com/
- Aer simulator: https://qiskit.github.io/qiskit-aer/
- PyVista: https://docs.pyvista.org/
- Distilled notes: `~/薦家/布球/` (Bloch sphere derivations, API references)
- viz-dboard: Streamlit rules and demo tool conventions
- Logic check: 2026-11-22 (rev-skill) — removed stray `EOF`; added Fredkin, mutual-exclusion pointer, `horizontal=True`. Second pass: **removed the incorrect matplotlib-3D exception** (it contradicted `viz-dboard`); 3D is now unconditionally PyVista + Panel, backed by the working `5布球態演化維問.py` implementation. Third pass: 2D narrowed to matplotlib only (no seaborn/PIL). Fourth pass: `qbit-gate-viz` Bloch panel converted from matplotlib 3D to 2D projection — no matplotlib-3D demo remains. Fifth pass (doc layer): removed matplotlib-backed Qiskit renderer examples (`plot_bloch_multivector` / `plot_state_qsphere` / `plot_state_city`); both demo code layers now comply (step 9 hook). Sixth pass (2026-11-22, review + implementation): `description` now starts with `Use when...`; added **Core Invariant**; `bloch-evolution-viz` rewritten to compute states with Qiskit `Statevector` and draw with PyVista + Panel, fixed the evolution off-by-one (previous code applied U^(frames+1)), added `Player` animation and α/β readout; `qbit-gate-viz/__init__.py` synced byte-for-byte with `app.py`, legacy `app.py.bak` (matplotlib-3D) removed, `__pycache__` purged. Seventh pass (user rule via `rev-skill`): 3D evolution folded **into the single `qbit-gate-viz/app.py`** (PyVista → Panel VTK pane → `st.iframe`); Core Invariant, all render references, Guardrails and Demo Tools updated to one integrated app; `bloch-evolution-viz` marked retired; added a pointer to `viz-dboard`'s distilled-first prerequisite instead of duplicating the rule.