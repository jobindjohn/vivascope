# vivascope — Specification

Status: draft v0.1 · Last updated: 2026-11-13

vivascope is a Python package that turns VIVA+ human body model (HBM) simulation
results from LS-DYNA into reports with plots and injury criteria. You can use it
from the command line, from Python, or (later) from a GUI. It is published on PyPI.

It continues the `dynapost` prototype. dynapost's processing features carry over,
and the known dynapost bugs (see §12) must not.

---

## 1. Goals and non-goals

### Goals (v0.1)

- Get a quick look at the results of one or more VIVA+ simulations.
- Compare any number of simulations (1 to N) in a single report.
- Compute the VIVA+ injury criteria using the definitions shipped with the VIVA+
  model (Dynasaur `.def` files).
- Produce a shareable static HTML report with interactive plots, and export the
  processed data to CSV.
- Ship an example marimo notebook that uses the public API for interactive
  exploration (§8.3).
- Install with `pip install vivascope`.

### Non-goals (v0.1)

- Reading `d3plot` or the separate ASCII output files (only `binout` and `d3hsp`).
- Reading `glstat`, or the global-energy quality checks in VIVA+ `02_*.def`.
- BrIC and chest deflection. VIVA+ does not define them yet, so they are deferred.
- Unit conversion beyond simple per-simulation scale factors (see §6).
- A GUI. One is planned later, and the API must not prevent it (see §8).
- A supported live app. The example notebook (§8.3) is the only interactive
  front end in v0.1.
- Verified agreement with Dynasaur output. This is a goal but not a release gate
  (see §11).

## 2. Users

The VIVA+ community: anyone running the open-source VIVA+ HBM
(<https://openvt.eu/fem/viva/vivaplus>) in LS-DYNA.

## 3. Inputs

| Input | Source | Content used |
|---|---|---|
| `binout*` | LS-DYNA, read with `lasso-python` | `matsum`, `rcforc`, `nodout`, `elout` |
| `d3hsp` | LS-DYNA text output | run info, part masses |
| Dynasaur `.def` files | VIVA+ repo `model/postprocess/Dynasaur/` | named IDs, criteria definitions |
| Config | YAML file + CLI overrides | everything else (§7) |

### 3.1 binout sections

| Section | Channels | Used for |
|---|---|---|
| `matsum` | `internal_energy`, `hourglass_energy`, `mass` (added mass), `legend`, `legend_ids` | energy plots, HG/IE ratio, added-mass ratio |
| `rcforc` | `x_force`, `y_force`, `z_force`, `legend`, `legend_ids` | contact force plots |
| `nodout` | `x/y/z_displacement`, `x/y/z_acceleration`, `rx/ry/rz_acceleration`, `time`, `legend`, `legend_ids` | displacement plots, HIC15, DAMAGE, NIC |
| `elout` | shell/solid strain with element → part mapping | rib strain, rib fracture risk |

VIVA+ only writes `elout` for body regions switched on in
`vivaplus-91-output-settings.k` (`HE_elout`, `TX_elout`, `LX_elout` are on by
default; `NE_elout`, `UX_elout`, `PE_elout` are off). If a section a criterion
needs is missing, vivascope skips that criterion and reports a warning (§9).

### 3.2 d3hsp

Parsed exactly as in dynapost:

- run date and time, LS-DYNA version and revision, number of CPUs, total CPU
  time, elapsed time, normal termination;
- total mass of each part, and of each rigid-body material.

## 4. VIVA+ definitions (Dynasaur `.def` files)

vivascope reads VIVA+'s Dynasaur `.def` files directly and treats them as the
source of truth. It does not keep its own copy of IDs or criteria parameters.

### 4.1 Where the files come from

- **Bundled default:** a copy of the `.def` files from the VIVA+ repository ships
  inside the package. It is pinned to a recorded VIVA+ commit; the initial pin is
  `6ae70cf` (v2.1.0rc2).
- **User override:** config key or CLI option pointing to a folder of `.def`
  files. This must work with any VIVA+ release that ships `.def` files.
- The report states which `.def` source was used: bundled (with its VIVA+
  version) or a user path.
- **Licensing:** the VIVA+ files are LGPL-3.0 and vivascope is Apache-2.0. The
  bundled files keep their LGPL-3.0 license text and notices in a separate
  directory. Their compatibility must be confirmed before the first PyPI release
  (§14).

### 4.2 What is read from them

| File | Used for |
|---|---|
| `00_vivaplus_IDs.def` | named `NODE` objects (266 landmarks), `OBJECT` part groups (110 body-region groups), `DISCRETE`, `ENERGY_PART` |
| `01_units.def` | read and shown in the report only; units are assumed to be ms / mm / kg (§6) |
| `10_head.def` | HIC15, DAMAGE, DAMAGE AIS1/AIS2/AIS4+ risk |
| `20_neck_injury_criteria.def` | NIC curve, NIC max, NIC risk |
| `40_Rib_Criteria.def` | rib strain → rib fracture risk (Weibull), 25/50/75 YO |
| `41_Rib_Criteria_erf_rib_risk.def` | rib strain → rib fracture risk (erf, Larsson et al. 2021), 25/50/75 YO |
| `ls_user_function.py` | reference for `erf_rib_risk_age`. vivascope reimplements it and does not execute this file. |

Other `.def` files (`02_*`, `03_contact`, `04_vehicle`, `11_*`, `21_*`, `70_*`,
`91_*`, `92_*`, `93_*`) are outside v0.1.

### 4.3 Definition interpreter

A criterion in a `.def` file is a tree of named functions whose leaves are data
references such as `{"type": "NODE", "ID": "<name>", "array": ["(0, x_acceleration)"]}`.
vivascope evaluates these trees.

- **Data references to resolve:** `NODE` → `nodout`, `OBJECT` → `elout` (strain
  per part group).
- **Functions to support in v0.1** (everything the chosen criteria files use):

  | Function | Used by |
  |---|---|
  | `cfc` (SAE J211 CFC filter) | HIC15, NIC |
  | `res` (resultant) | HIC15 |
  | `max` | HIC15, NIC |
  | `HIC_15` | HIC15 |
  | `DAMAGE` | DAMAGE |
  | `extended_ccdf_weibull` | DAMAGE AIS risks |
  | `NIC` | NIC |
  | `ccdf_sigmoid` | NIC risk |
  | `stress_strain_part_max` | rib strain |
  | `mult` | rib risk (Weibull) |
  | `ccdf_weibull` | rib risk (Weibull) |
  | `erf_rib_risk_age` | rib risk (erf) |
  | `linspace`, `binom` | overall / individual rib-fracture risk (number of fractured ribs) |

- If a criterion uses a function or data type the interpreter does not support,
  that criterion is skipped with a warning. It must not stop the report.
- **Licensing constraint:** Dynasaur is GPL-3.0. vivascope must not copy
  Dynasaur code. Each function is implemented from its published method (e.g.
  SAE J211 for CFC, the cited papers for risk curves) and documented with the
  reference.
- The `.def` files also contain `DATA VISUALIZATION` sections. Whether vivascope
  uses them is an open question (§14).

### 4.4 Model variants

The four VIVA+ models (50F/50M, seated/standing) share all IDs and differ only
in node coordinates and the `SEX` parameter. In v0.1, "model variant" is a
**label only**: each simulation can be tagged with its variant, and that tag is
used in report labels and comparisons. Variants do not change how anything is
computed.

## 5. Processing features (carried over from dynapost)

Per data section (`matsum`, `rcforc`, `nodout`):

- **Time filtering:** `time_min`, `time_max`.
- **ID filtering:** `id_min`, `id_max` (ID range), and for `nodout` an explicit
  ID list or a CSV file of IDs.
- **Top-N selection:** show only the N curves with the largest peak.
- **Per-simulation scaling:** `time_scale`, `mass_scale`, `force_scale`,
  `energy_scale`.
- **Per-simulation ID offset:** `id_offset`, applied to ID filters and ID lists.
- **rcforc mode:** master (`m`), slave (`s`), or `both`.
- **Labels:** curves are labelled `ID_name` or `name_ID` (`id_first`).
- **Derived quantities:** HG/IE ratio per part, added mass / total mass ratio
  per part (total mass from d3hsp), total-mass summary (all parts and filtered
  parts).
- **CSV export** of the processed data, to the simulation folder by default or
  to a configured folder.

In addition, VIVA+ named landmarks from `00_vivaplus_IDs.def` can be used in
place of raw node IDs for `nodout` selection, and VIVA+ part groups (`OBJECT`) in
place of raw ID ranges.

## 6. Units

- vivascope assumes VIVA+ units: **ms, mm, kg**. Derived units are kN, kJ,
  mm/ms², and so on.
- The scale factors in §5 are the only conversion. They cover simulations run in
  other units, as in dynapost.
- Criteria are computed after scaling, so their inputs are always in VIVA+
  units.

## 7. Configuration

- **Format:** YAML.
- **Overrides:** repeatable `--set dotted.key=value` on the CLI, as in dynapost.
  The value parser must handle `null`, `true`/`false`, numbers and strings
  consistently. dynapost's `None`/`null` mismatch (§12) must not carry over.
- **Schema:** typed and validated at load time. Unknown keys are errors, which
  prevents dynapost's silently ignored keys (§12).
- **Defaults:** a `defaults` section whose values apply to every data section
  unless the section overrides them. In dynapost they were documented but never
  applied.
- **Per-simulation overrides:** a list of simulations, each with `path`,
  `label`, `variant`, scale factors, ID offsets, and section-level overrides.
- **No machine-specific paths** in shipped example configs.
- vivascope ships a default config. Running with no config file must produce a
  sensible report.

## 8. Interfaces

### 8.1 CLI

- One command that takes 1 to N simulation folders and produces a report. A
  single-simulation report is the N = 1 case, so there is no separate compare
  mode.
- Options: config file, `--set` overrides, output path, `.def` folder override,
  CSV export toggle.
- Exact command names and flags: open question (§14).

### 8.2 Python API

- Public functions for: loading a simulation, computing sections and criteria,
  building a report, and exporting data. Each step can be called on its own so
  that notebooks, scripts and a future GUI can reuse it.
- Results are returned as plain data (pandas DataFrames / dataclasses), separate
  from plotting. A future GUI then needs no changes to the core.

### 8.3 Example notebook

- A marimo notebook in `examples/` with widgets for changing filters and
  selections and seeing the results update.
- It uses only the public API (§8.2). If it needs anything else, the API is
  missing something and must be fixed in the core, not in the notebook.
- It is an example, not a supported interface: no stability promise, and it is
  not installed with the package. marimo is not a vivascope dependency.
- A supported live app and its technology are decided after v0.1, together with
  the GUI (§14).

### 8.4 GUI

Planned after v0.1. Not specified here.

## 9. Outputs

### 9.1 Static HTML report

- A single self-contained file that can be shared, with interactive plots
  (zoom, hover, toggle curves). Which plotting library to use is an open
  question (§14).
- **Sections:**
  1. Header: simulations included, with label, path, model variant and `.def`
     source.
  2. Warnings: everything skipped or failed, prefixed by simulation label.
  3. Run info: per-simulation table from d3hsp.
  4. Mass: total and filtered mass, number of parts, added-mass ratio plot.
  5. Energies: internal, hourglass, and HG/IE ratio.
  6. Contact forces.
  7. Node displacements.
  8. Injury criteria: HIC15; DAMAGE with AIS risks; NIC with risk; rib strain
     and rib fracture risk (Weibull and/or erf, chosen in config, at 25/50/75 YO).
     Scalar results go in tables with one column per simulation; time histories
     (e.g. NIC curve, filtered head acceleration) are plotted.
- **Multiple simulations:** curves from different simulations are told apart by
  line style or similar, and use the same colours for the same IDs.
- Missing data must never fail the whole report. Each section fails on its own
  and adds a warning.

### 9.2 CSV / data export

All processed DataFrames, and a table of criteria results, written as CSV.

## 10. Architecture and dependencies

- Plain Python package with a `src/` layout. Notebooks are not used as library
  code (unlike dynapost's marimo `functions.py`).
- Layers kept separate: **I/O** (binout, d3hsp, `.def`) → **processing**
  (filtering, scaling, top-N, derived quantities) → **criteria** (`.def`
  interpreter and functions) → **presentation** (plots, HTML report). The
  example notebook sits outside the package and calls the public API.
- **Python:** follows `lasso-python`. As of `lasso-python` 2.0.4 that is
  Python ≥ 3.11 with numpy > 2.2.
- Dependencies are declared exactly as used. dynapost's undeclared `pyyaml` and
  unused `plotnine` and `pyarrow` (§12) must not carry over.
- Dynasaur is **not** a dependency (GPL-3.0, and it pins `lasso-python==1.5.0`).

## 11. Testing and validation

- **Unit tests** with synthetic data for: parsers (d3hsp text, legend parsing),
  filtering, scaling, top-N, the config loader and overrides, and every
  interpreter function against hand-computed or published reference values (CFC
  filter, HIC15, NIC, DAMAGE, risk curves).
- **`.def` parsing tests** against the bundled `.def` files.
- **Real simulation data** is too large to publish and stays local. Integration
  tests that need it are skipped in CI unless an environment variable points to
  local reference results.
- **Dynasaur comparison** (nice to have): a local script that runs Dynasaur in a
  separate environment on the same reference simulations and compares criteria
  values with vivascope's, within stated tolerances.
- **Example notebook:** CI runs it headless on synthetic data so that API
  changes that break it are caught.

## 12. dynapost issues that must not carry over

| dynapost issue | Requirement in vivascope |
|---|---|
| `warning_list` typo crashes the single report when matsum is disabled | Each section is isolated; tests cover disabled sections |
| Compare mode applies `mass_scale` to added mass but not to total mass, so the ratio is wrong | Scaling is applied consistently to both values of every ratio |
| Top-N ranks by signed max, so negative forces are under-ranked | Top-N ranks by absolute peak (rcforc: resultant) |
| Top-N picked separately per x/y/z direction | One selection per entity, shared across directions |
| `None` vs `null` mismatch in per-sim overrides | One value parser (§7) |
| Compare `intersect` mode matches on labels, so it breaks with ID offsets | Matching is on IDs after offset |
| File read errors are printed, not reported | All errors go to the report warnings |
| Config keys documented but ignored (`defaults.*`, `rcforc.show_top`, `nodout.extract_absolute`) | Validated schema; every key has an effect |
| Library code inside a marimo notebook, imported via `sys.path` | Plain package (§10) |

## 13. Project, release and docs

- **Repository:** GitHub (`vivascope`).
- **CI:** GitHub Actions runs lint and tests on the supported Python versions.
- **Release:** PyPI trusted publishing from GitHub Actions on a version tag.
- **License:** Apache-2.0 for vivascope. Bundled VIVA+ `.def` files keep LGPL-3.0
  (§4.1).
- **Authors:** the original dynapost authors are listed as authors in
  `pyproject.toml` and `CITATION.cff`. **TODO: names.**
- **Docs:** README only for v0.1. It covers installation, a quickstart (CLI and
  API), a config reference, the list of criteria with their references, how to
  run the example notebook, and how to cite.

## 14. Open questions

1. Plotting library for interactive plots (plotly, Bokeh, Altair, …).
2. Technology for the supported live app / GUI after v0.1 (marimo, Streamlit,
   Dash, …). Not needed for v0.1.
3. CLI command names and flags.
4. Should the `.def` `DATA VISUALIZATION` sections drive plots in the report?
5. Is bundling LGPL-3.0 `.def` files in an Apache-2.0 package acceptable as
   specified in §4.1, and what notices are required?
6. Author names for `pyproject.toml` / `CITATION.cff`.
7. Tolerances for the Dynasaur comparison (§11).
8. BrIC and chest deflection: definitions and source, for a release after v0.1.
