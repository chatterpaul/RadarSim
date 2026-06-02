# Local Windows Setup Notes

Date validated: 2026-05-31

Repository: `C:\Users\chatt\Documents\Codex\RadarSim-reference-codex-assessment`

Branch: `codex/windows-setup-notes`

## Scope

This note records a local Windows setup and execution smoke test for RadarSim. It uses only the local RadarSim clone and public dependencies from `requirements.txt`.

## Python Version

Recommended Windows version for repeatable setup: Python 3.11, or Python 3.10 if 3.11 is unavailable.

Rationale:

- `README.md` advertises Python 3.10+.
- `pyproject.toml` classifiers cover Python 3.9 through 3.12, and the mypy config targets Python 3.10.
- Python 3.13 is not declared in the project metadata, even though the latest dependency wheels installed and passed tests during this validation.

Version actually used for this run:

```powershell
Python 3.13.5
```

Python launcher discovery showed a Python 3.10 WindowsApps registration, but it was not usable:

```powershell
py -0p
py -3.10 --version
```

Result:

```text
 -V:3.13 *        C:\Python313\python.exe
 -V:3.10          C:\Program Files\WindowsApps\PythonSoftwareFoundation.Python.3.10_3.10.3056.0_x64__qbz5n2kfra8p0\python3.10.exe
Unable to create process ... Access is denied.
```

No `conda`, `mamba`, `micromamba`, `uv`, `pyenv`, or `python3.11` executable was available on PATH.

## Commands Used

Initial repository guardrail checks:

```powershell
pwd
git remote -v
git branch --show-current
git status
```

Environment discovery:

```powershell
py -0p
py -3.10 --version
python --version
where.exe python
where.exe conda
where.exe mamba
where.exe micromamba
where.exe uv
where.exe pyenv
where.exe python3.11
```

Virtual environment and dependency install:

```powershell
python -m venv .venv
.venv\Scripts\python --version
.venv\Scripts\python -m pip --version
.venv\Scripts\python -m pip install -r requirements.txt
.venv\Scripts\python -m pip check
.venv\Scripts\python -m pip freeze
```

Validation commands:

```powershell
.venv\Scripts\python -m pytest -q
.venv\Scripts\python headless.py --help
.venv\Scripts\python batch_run.py --help
.venv\Scripts\python headless.py --range 30 --rcs 1.0 --duration 1
.venv\Scripts\python headless.py --config scenarios\f16_vs_sa6.yaml
.venv\Scripts\python batch_run.py --quick --configs 2 --runs 1 --workers 1 --output output\batch_smoke.csv
.venv\Scripts\python -u run_gui.py
.venv\Scripts\python examples\api_examples.py
.venv\Scripts\python -c "import pyqtgraph.opengl"
```

Dependency import/version smoke:

```powershell
.venv\Scripts\python -c "import sys, numpy, scipy, numba, PySide6, pyqtgraph, yaml, sklearn, pandas; print(sys.version); print('numpy', numpy.__version__); print('scipy', scipy.__version__); print('numba', numba.__version__); print('PySide6', PySide6.__version__); print('pyqtgraph', pyqtgraph.__version__); print('yaml', yaml.__version__); print('sklearn', sklearn.__version__); print('pandas', pandas.__version__)"
```

## Environment Type

A local venv was used:

```powershell
.venv
```

No conda or mamba environment was used.

## Dependency Install Results

`pip install -r requirements.txt` succeeded after allowing network access. The first sandboxed attempt failed with Windows socket permission error `[WinError 10013]`; rerunning with approved network access installed all requirements.

`pip check` result:

```text
No broken requirements found.
```

Key installed versions:

```text
numpy==2.4.6
scipy==1.17.1
matplotlib==3.10.9
numba==0.65.1
llvmlite==0.47.0
PySide6==6.11.1
pyqtgraph==0.14.0
PyYAML==6.0.3
scikit-learn==1.8.0
pandas==3.0.3
pytest==9.0.3
```

Numba installed successfully. The previous missing-Numba blocker is resolved in this venv.

## Pytest Result

Command:

```powershell
.venv\Scripts\python -m pytest -q
```

Result:

```text
217 passed, 1 warning in 11.47s
```

The only warning was a pytest cache write warning:

```text
PytestCacheWarning: could not create cache path ... .pytest_cache\v\cache\nodeids: [WinError 5] Access is denied
```

This was a local cache write warning, not a test failure.

## Headless Result

Help command:

```powershell
.venv\Scripts\python headless.py --help
```

Result: succeeded and printed CLI usage.

Single-run smoke:

```powershell
.venv\Scripts\python headless.py --range 30 --rcs 1.0 --duration 1
```

Result:

```text
Pulses transmitted: 990
Detections: 0
Detection Probability (Pd): 0.000
Mean SNR: -3.0 dB
Runtime: 284.7 ms
```

Scenario smoke:

```powershell
.venv\Scripts\python headless.py --config scenarios\f16_vs_sa6.yaml
```

Result:

```text
Pulses transmitted: 9,999
Detections: 9,999
Detection Probability (Pd): 1.000
Mean SNR: 36.4 dB
Runtime: 149.7 ms
```

## Batch Result

Help command:

```powershell
.venv\Scripts\python batch_run.py --help
```

Result: succeeded and printed CLI usage.

Initial batch smoke command:

```powershell
.venv\Scripts\python batch_run.py --quick --configs 2 --runs 1 --workers 1 --output output\batch_smoke.csv
```

Initial result: simulations completed, then Windows console encoding failed on the success checkmark:

```text
UnicodeEncodeError: 'charmap' codec can't encode character '\u2713'
```

Fix applied: `batch_run.py` now reconfigures stdout/stderr to UTF-8 with replacement, matching the existing compatibility pattern in `run_gui.py`.

Retest result:

```text
Results saved to: output\batch_smoke.csv
BATCH COMPLETE
Total simulations: 2
Total pulses: 19,998
Total detections: 19,998
Average Pd: 1.000
Average SNR: 20.3 dB
Total time: 1.47s
Sims/second: 1.4
```

## GUI Result

Command:

```powershell
.venv\Scripts\python -u run_gui.py
```

Result: dependency checks passed and the app reached GUI startup. The process stayed alive until the 20-second guard timeout, which is expected for a Qt event loop in this non-interactive validation.

Captured output:

```text
✓ PySide6 OK
✓ PyQtGraph OK
✓ PyYAML OK
✓ Numba OK
✓ Physics engine OK

Starting GUI...
[WARNING] PyQtGraph OpenGL not available - 3D view disabled
[SETTINGS] Loaded saved configuration
```

Known GUI limitation: the 3D view is disabled because `pyqtgraph.opengl` requires `PyOpenGL`, which is not listed in `requirements.txt`.

Confirming command:

```powershell
.venv\Scripts\python -c "import pyqtgraph.opengl"
```

Result:

```text
ModuleNotFoundError: No module named 'OpenGL'
```

## Examples Result

`examples/api_examples.py` appears stale relative to the current `src/` package layout.

Command:

```powershell
.venv\Scripts\python examples\api_examples.py
```

Result:

```text
ModuleNotFoundError: No module named 'radar_physics'
```

The example imports old top-level modules such as `radar_physics`, `target_tracking`, and `ecm_simulation`; current tests and scripts import from `src.*`.

## Inspected Files And Directories

Inspected setup and entry files:

- `README.md`
- `requirements.txt`
- `pyproject.toml`
- `run_gui.py`
- `headless.py`
- `batch_run.py`

Inspected validation assets:

- `tests/`
- `examples/`
- `scenarios/`

Scenario files present:

- `scenarios/basic_tracking.json`
- `scenarios/close_air_combat.yaml`
- `scenarios/drone_swarm_saturation.yaml`
- `scenarios/ecm_environment.json`
- `scenarios/f16_vs_sa6.yaml`
- `scenarios/ground_clutter_filtering.yaml`
- `scenarios/hypersonic_interception.yaml`
- `scenarios/mountain_ambush.yaml`
- `scenarios/naval_battlegroup.yaml`
- `scenarios/stealth_deep_penetration.yaml`

Test categories present:

- physics core and validation
- signal processing and pulse-Doppler
- simulation objects and engine
- EKF/tracking
- ECCM
- network fusion
- Phase 30 SAR/ISAR and AI director

## Dependency And Metadata Drift

Observed drift:

- `README.md` and docs mention PyQt6, but the actual GUI code and `requirements.txt` use PySide6.
- `pyproject.toml` has `PyQt6` as the GUI extra, while the code imports PySide6.
- `pyproject.toml` core dependencies include `pygame`, but `requirements.txt` does not.
- `requirements.txt` includes required runtime/test packages missing from `pyproject.toml`, including `numba`, `PySide6`, `pyqtgraph`, `h5py`, `pyyaml`, `scikit-learn`, `joblib`, `pandas`, `tqdm`, and `pytest`.
- 3D GUI support needs `PyOpenGL`, but it is not included in `requirements.txt`.
- Dependency lower bounds are broad and unpinned, so fresh installs may vary substantially over time.

## Known Blockers

- The local Python 3.10 WindowsApps interpreter is registered but unusable due `Access is denied`.
- `examples/api_examples.py` fails because it imports stale top-level module names.
- GUI 3D view is disabled without `PyOpenGL`.
- Project metadata does not declare Python 3.13 support, although this validation passed on Python 3.13.5 with latest wheels.
- `.pytest_cache` produced a local access-denied warning during tests.

## Recommended Fixes

- Update `pyproject.toml` to match the real runtime stack, especially PySide6, Numba, pyqtgraph, PyYAML, scikit-learn/pandas/joblib/tqdm, and test extras.
- Decide whether Python 3.13 is supported. If yes, add classifier/test coverage; if no, document Python 3.10/3.11 as the recommended Windows versions and constrain dependencies accordingly.
- Add `PyOpenGL` to an optional GUI/3D extra or to `requirements.txt` if the 3D tactical view should work out of the box.
- Refresh `examples/api_examples.py` to import from the current `src.*` modules or remove/update stale examples.
- Consider pinning or bounding dependencies for a reproducible Windows setup.
- Keep the small `batch_run.py` Windows console encoding fix.

## PR Recommendation

A personal fork PR is recommended for:

- `docs/LOCAL_WINDOWS_SETUP_NOTES.md`
- the small `batch_run.py` Windows console encoding compatibility fix

An upstream PR is also reasonable if kept narrow. For upstream, separate the documentation/encoding fix from larger dependency metadata cleanup so maintainers can review the low-risk Windows validation work independently.
