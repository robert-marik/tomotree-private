# Ponytail audit (over-engineering) – 2026-10-05

Scope: dead code, unused dependencies, needless abstractions. Nothing has been applied yet.
Ranked by size of the cut.

| # | Tag | What to cut | Replacement | Where |
|---|-----|-------------|-------------|-------|
| 1 | delete | Commented-out old code, ~54 lines (e.g. old `process_data`) | Git history | `src/tomotree/streamlit/tomotree.py:33-120` |
| 2 | delete | Commented-out blocks, 80 comment lines | Git history | `src/tomotree/streamlit/resistograph/drilling_path.py` |
| 3 | delete | Legacy persistent widget wrappers, imported nowhere (CLAUDE.md: legacy) | Widget registry | `src/tomotree/streamlit/widgets/persistent_widgets.py` (176 lines) |
| 4 | delete | Duplicate polygon second moment, imported nowhere | `SectionProperties.moment_of_inertia_by_tensor` | `src/tomotree/physics/momentum_tensor.py` (139 lines) |
| 5 | delete | Legacy .mo translation compiler, not used at runtime (`docs/i18n.md`) | `make json` | `src/tomotree/i18n/build_translations.py` (215 lines) |
| 6 | delete | Unused module, mostly commented-out code plus a live import and storage status call | – | `src/tomotree/streamlit_app/user.py` (59 lines) |
| 7 | delete | `get_asset_path` with three fallbacks, no caller | `importlib.resources.files("tomotree")` if ever needed | `src/tomotree/streamlit/assets.py` (49 lines) |
| 8 | delete | `slider_with_state`, no caller, a fourth way to keep widget state | Registry `sliders` | `src/tomotree/streamlit/slider.py` (34 lines) |
| 9 | delete | Dependency `scikit-learn`: nothing imports `sklearn`, only listed on the health page | Drop from requirements and `streamlit_app/health.py:41` | `requirements-mamba.txt` |
| 10 | delete | Dependency `openpyxl`: no Excel read/write anywhere | – | `requirements-mamba.txt` |
| 11 | delete | Dependency `markdown`: never imported (`st.markdown` does not need it) | – | `requirements-mamba.txt` |
| 12 | delete | Dependency `httpx`: never imported directly | Drop it if `st.login` still works without it (not verified) | `requirements-mamba.txt` |
| 13 | yagni | `rich` only for the one-off script `src/analyze_f3d.py` | Move script to `demo/` or a dev-only extra | `src/analyze_f3d.py` |
| 14 | yagni | `seaborn` for a single plot | Plain matplotlib / pandas `.plot` | `src/tomotree/streamlit/showdata.py` |
| 15 | delete | Legacy helpers `make_pills()` / `get_text_input_params()` (still called from 7 pages) | Move those pages to the registry, then delete | `src/tomotree/streamlit/widgets/__init__.py` |

**net: about -900 lines, -4 deps possible** (`scikit-learn`, `openpyxl`, `markdown`, `httpx`), plus `rich` and `seaborn` if moved or rewritten.

## Checked and kept

- `tabulate`: needed by pandas `to_markdown` (snippet tables in `tomogram.py`, `resistograph/plot.py`).
- `src/tomotree/demo_data/demo_data.py`: used by `demo/overview.py`.
- `_try_groq_fix` in `config_editor.py`: optional, but called 4 times.

## Not over-engineering, but worth noting

- `src/tomotree/streamlit_app/config_editor.py` (1862 lines, biggest file): the pure layer functions
  (`list_layers`, `get_layer`, `add_layer`, `update_layer`, `delete_layer`, `switch_to_*_layer`)
  belong in the library, not the GUI. This is a placement question (architect agent), not a deletion.
