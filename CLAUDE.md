# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

**Install dependencies:**
```bash
pip install -r requirements-dev.txt
# or
pip install .[dev]
```

**Run tests:**
```bash
pytest test/
```

**Run a single test file:**
```bash
pytest test/test_analyzer.py
```

**Run a single test:**
```bash
pytest test/test_analyzer.py::TestBeamAnalyzer::test_get_internal_shear
```

**Run with coverage:**
```bash
coverage run -m pytest
coverage report -m
```

**Type checking:**
```bash
mypy pybeam/
```

**Lint:**
```bash
pylint pybeam/
```

**Build package:**
```bash
python -m build
```

## Architecture

PyBeam is a structural beam analysis library. The data flows through these layers:

1. **Loads** (`loads.py`) — Abstract `Load` hierarchy. Concrete types:
   - `PointForce` — shear or axial point load, position normalized 0–1
   - `PointMoment` — point moment, position normalized 0–1
   - `UniformDistributedLoad` — distributed load over a normalized range

2. **LoadingCase** (`loading_case.py`) — Data container grouping all loads on a beam (axial, shear, point moments, torsional) plus the discretized point array (`np.linspace`). Positions within loads are always **normalized** (0 to 1); the `LoadingCase` holds the real `length` and resolves positions.

3. **BeamAnalyzer** (`analyze.py`) — Stateless computation on a `LoadingCase`. Computes internal forces via `np.cumsum` over each load's `load_distribution()` output. Does not know about material or profile properties.

4. **Members** (`members.py`) — User-facing API:
   - `Loadable` — can be instantiated directly; wraps a `LoadingCase` and exposes `add_shear_point_force`, `add_axial_point_force`, `add_point_moment`. Methods return `self` for chaining. Exposes `analyze()` and `plot()`/`save_plot()`.
   - `UniformMember(AbstractMember, Loadable)` — combines `Loadable` with a `StaticProfile` and `Material` for weight and section property queries.

5. **Profiles** (`profiles.py`) — Cross-section geometry. `StaticProfile` ABC with `get_area()` and `get_area_moment_of_inertia()`. `IBeamProfile` is the only concrete implementation.

6. **Materials** (`materials.py`) — `Material` dataclass with density, modulus, strengths. `Steel` (A36) is the only predefined material.

7. **Visualizers** (`visualizers.py`) — `Visualizer` ABC with `render(analyzer)`. `MatplotlibVisualizer` renders a 3-panel figure (load diagram, shear diagram, bending moment diagram). Called via `analyzer.visualize(vis)`.

**Key convention:** All load positions passed to `PointForce`, `PointMoment`, and `UniformDistributedLoad` must be **normalized** (0 to 1 fraction of beam length), not absolute positions. The `Loadable` methods accept absolute positions and divide by length before creating load objects — but the raw load classes assert `0 <= position <= 1` directly.

**Sign conventions:**
- Positive shear force: acts downward
- Positive moment: causes sagging (concave up)
