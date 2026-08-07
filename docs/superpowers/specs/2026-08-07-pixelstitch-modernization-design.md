# pixelstitch modernization — design

**Date:** 2026-08-07
**Goal:** pixelstitch works on modern stacks (Python 3.12+, matplotlib 3.10, numpy 2, notebook 7 / JupyterLab / VS Code) *and* on the old known-good stack (Python 3.9, matplotlib 3.4, numpy 1, classic notebook 6), with CI guarding both.

## Context

- The repo is an nbdev **v1** project. The generated `pixelstitch/` package was never
  committed, so installing from a source checkout fails.
- `settings.ini` pins `matplotlib<3.5` and `numpy<2` (workarounds from merged PR #2,
  Aug 2024).
- CI (`.github/workflows/main.yml`) uses removed nbdev v1 commands on Python 3.7 — dead.
- **PR #3** (draft, by Dawars) is a mostly-complete nbdev v2 migration with open TODOs
  (docs pipeline, CI, Pages permissions).
- **Issue #1**: clicks only register with `matplotlib<3.5` + `notebook<7` + manual
  nbextension setup. **Issue #4**: `%matplotlib ipympl` broken in VS Code.

Root causes of modern-matplotlib breakage in `core.ipynb`:

1. `process_user_click` gates on `str(self.figure.canvas.toolbar.cursor)` — the
   `cursor` attribute changed/disappeared across matplotlib/ipympl versions, and
   `toolbar` can be `None`, so clicks silently stop registering.
2. `ax.text([], [], '')` — empty-list positions raise on draw in modern matplotlib.
3. `%matplotlib notebook` (nbagg) does not exist in notebook 7 / JupyterLab / VS Code.
4. `from IPython.core.display import display, HTML` — removed in modern IPython.
5. `numpy<2` pin is obsolete: kornia and opencv support numpy 2 now.

## Design

### 1. Base: finish PR #3 (nbdev v2 migration)

Fetch `Dawars:update_nbdev`, merge current `master` into it (master moved since
Aug 2024), resolve conflicts, preserve Dawars' authorship in history. This brings
quarto docs config, nbdev v2 workflows, and commits the generated `pixelstitch/`
package so source installs work.

### 2. Version-adaptive widget code (`core.ipynb`)

- Replace the `toolbar.cursor` click-guard with a portable helper:
  - `toolbar is None` (Agg / headless) → allow the click;
  - otherwise require `toolbar.mode == ''` (idle) — true on both classic
    NavigationToolbar2 (mpl 3.4) and modern ipympl toolbars.
- `ax.text(0, 0, '')` instead of `ax.text([], [], '')`.
- Import `display, HTML` from `IPython.display`.
- Docs/README: `%matplotlib widget` (ipympl) as the modern instruction;
  `%matplotlib notebook` stays documented for classic notebook 6 setups.
- Interactive demo cells get `#|eval: false` so CI never executes frontend-only magics.

### 3. Packaging

`settings.ini` requirements become `matplotlib>=3.4 kornia_moons kornia>=0.5.10 numpy`
— no upper bounds. Compatibility lives in the code, not in pins. `min_python`
stays as permissive as nbdev v2 allows.

### 4. Tests (nbdev-style, in notebooks)

Headless test cells in `core.ipynb`: force Agg backend, instantiate
`CorrespondenceAnnotator` on `sample_project`, call `start()`, then synthesize
matplotlib `MouseEvent`s:

- left-click inside `ax1` → point appended to `pts1`;
- right-click near that point → point removed;
- save → `corrs.txt` written with expected shape.

These run via `nbdev_test` in CI, so real interaction logic is guarded, not just imports.

### 5. CI/CD (GitHub Actions)

- `test.yaml`: two-leg matrix —
  - **old**: Python 3.9, `matplotlib~=3.4.0`, `numpy<2`;
  - **modern**: Python 3.12, latest matplotlib/numpy.
  Each leg: install package, notebook/library sync check (`nbdev_export` + git diff),
  `nbdev_test` with `MPLBACKEND=Agg`.
- `deploy.yaml`: nbdev v2 → quarto → GitHub Pages. One-time manual step: set
  repo Pages source to "GitHub Actions".
- No automated PyPI publishing (release via `nbdev_pypi` locally when wanted).

### 6. Aftermath

- Close PR #3 with credit once its content is merged.
- Reply on issue #1 with the new install story (`pip install pixelstitch` +
  `%matplotlib widget`).
- Reply on issue #4: VS Code needs `ipywidgets` + `ipympl` in the kernel env;
  verify behavior of the fixed code there.

## Error handling

- Headless/exotic backends (no toolbar) must not crash click handling — guard is
  part of the design (2).
- If ipympl is missing in a modern frontend, matplotlib's own magic error message
  is sufficient; no custom detection.

## Out of scope

- Auto-publishing to PyPI.
- JupyterLab-specific UI work beyond what ipympl provides.
- The commented-out tilt-slider feature.
