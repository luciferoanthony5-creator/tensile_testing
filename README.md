# Tensile Test Analysis

Analysis of tensile test data for 14 specimens (`A1`–`A7`, `S1`–`S7`) using Python, NumPy, pandas and matplotlib in Jupyter notebooks.

## Environment
- Python 3.13 in `.venv/`. Run notebooks with the `.venv (3.13.5)` kernel.
- Libraries used: `numpy`, `pandas`, `matplotlib`, `pathlib`, `scipy` (`scipy.stats.linregress`). Install new packages with `~/projects/tensile_test/.venv/bin/pip install <pkg>`.
- Paths in the notebooks are relative to this folder (`tensile_test/`).
- Plots open in a pop-up window (`%matplotlib qt`, via WSLg) instead of inline. To support this, `PyQt6` is installed in `.venv`, and Qt's system libraries were installed with apt:
  `libglib2.0-0t64 libegl1 libgl1 libfontconfig1 libfreetype6 libdbus-1-3 libwayland-client0 libwayland-cursor0 libx11-xcb1 libxkbcommon0 libxkbcommon-x11-0 libxcb-cursor0 libxcb-icccm4 libxcb-image0 libxcb-keysyms1 libxcb-randr0 libxcb-render0 libxcb-render-util0 libxcb-shape0 libxcb-shm0 libxcb-sync1 libxcb-util1 libxcb-xfixes0 libxcb-xkb1`

## Data
- `Data_export/*.csv`: raw test-machine exports, one per specimen, about 1000 rows each. Ignore the `*:Zone.Identifier` files (Windows download metadata).
  Columns: `Position (in), Force (lbf), Strain (%), Time (sec)`
- `data_with_stress/*.csv`: the same data plus a calculated stress column, written by `stres_clalc.ipynb`.

  | Column index | 0 | 1 | 2 | 3 | 4 |
  |---|---|---|---|---|---|
  | Name | Position (in) | Force (lbf) | Strain (%) | Time (sec) | Stress (psi) |

  Strain is in percent: divide by 100 for in/in.
- `diameters.csv`: headers `Specimen, d_o (in)`, one row per specimen. It keeps the plain `d_o (in)` header; only the exported results use `d₀`. Written by `stres_clalc.ipynb` and read by `material_properties.ipynb` cell 6. It's in `tensile_test/`, **not** `data_with_stress/`: `material_properties` cell 0 loads every `*.csv` in that folder as a specimen.
- `results_A.csv`, `results_S.csv`: the Minitab export, written by `material_properties.ipynb` cell 6.
- `plots/`: saved figures, one `<name>_yield.png` per specimen, written by `material_properties.ipynb` cell 4. Re-running the cell overwrites them.

## Notebooks

This section and the Plan record the purpose of each step and the reasoning behind it. The **Code walkthrough** section below explains the code as it stood on 2026-09-30. The notebooks are the source of truth for the current code.

### `stres_clalc.ipynb` (done): calculates stress
- Calculates `Stress (psi) = 4F / (πD²)` from each specimen's diameter `D` and writes the results to `data_with_stress/`.
- Diameters are typed in with `input()` for each specimen when the loop runs. The last run (2026-09-30) used the measured diameters, 0.495–0.503 in. They're also saved to `diameters.csv`.

### `material_properties.ipynb` (in progress): NumPy analysis
Finds E, 0.2% offset yield and UTS for every specimen, plots the curves, and exports the results for Minitab.

| Cell | Purpose |
|---|---|
| 0 | Load all 14 files, plot the stress–strain curves in one window |
| 1 | UTS |
| 2 | Young's modulus (plus A4's separate fit) |
| 3 | 0.2% offset line and yield strength, one window with 14 subplots |
| 4 | Same as cell 3, but 14 separate windows, each also saved to `plots/`. Run 3 **or** 4; both fill `yield_02` with the same values. |
| 5 | `plt.close("all")`: closes every plot window |
| 6 | Build the results table and export `results_A.csv` / `results_S.csv` |

## Data notes
- **A4:** the extensometer was not mounted correctly. Strain reads 0.153% for the first ~300 data points, so A4's E and 0.2% offset yield will be wrong. Keep A4 in the exported CSV; it can be removed in Minitab. The lab report should discuss the effect of removing vs. keeping it.
  A4's UTS (~63 ksi) is also well above the other A specimens (~47 ksi). UTS doesn't depend on the extensometer. With the measured diameter (0.498 in, close to the others) it's still ~63 ksi, so the diameter doesn't explain it. Check A4's material/temper.
- **Extensometer removed near UTS:** strain stops changing after ~0.9–1.1% for every specimen, so each stress–strain curve ends in a flat tail. Strain values are only valid up to UTS (the point of maximum stress).
- **Negative force in the first ~20 rows:** this is from before the test started (grip clamping / load cell zero offset), not real compression.
- **Negative starting strain (A1, S4, S7):** probably noise, or the extensometer was deflected after being zeroed. It may not have been re-zeroed after being mounted on the specimen.
- **Grip seating at the start:** A3 oscillates while increasing for the first ~90 points instead of rising steadily. S6 is nearly flat until ~0.02% strain.

## Plan
1. **Young's modulus:** fit a line to the linear (elastic) portion of each curve. The slope is E.
   - Lower strain bound: above 0 to drop the negative starting strain. Try ~0.02% to also skip the grip-seating curve in A3 and S6. Tune by testing.
   - Upper strain bound: ~0.2%. At 0.2% strain every specimen except A4 is within ~1% of the fitted line, and yield is at ~0.6%, so this stays in the elastic region. Tune by testing. This 0.2 is NOT the 0.2% offset.
   - Strain is stored in %. Convert to in/in before fitting, or E comes out 100× too small.
2. **Stress–strain curve** for each specimen, on its own plot.
3. **Elastic-region plot** for each specimen: stress vs. strain over the fit window only. The stress and strain points must stay paired, so both are cut to the same rows.
4. **0.2% offset line** for each specimen: `stress_offset = E·(ε − 0.002) + b`.
   - Use the full strain range, not just the fit window. Yield is at ~0.6% strain, outside the window, so the line must extend past it to cross the curve.
   - 0.002 is in/in, so ε must be in/in too.
5. **Yield strength:** where the offset line crosses the stress–strain curve, i.e. where `stress − stress_offset` changes sign from + to −.
   - Search only up to UTS. After UTS the strain is frozen, so any crossing there is meaningless.
   - Handle the case where there is no crossing.
6. **Export CSV for Minitab:** two files, `results_A.csv` (A1 … A7) and `results_S.csv` (S1 … S7), one row per specimen. Headers: `Specimen, d₀ (in), A₀ (in²), E (Msi), R², Yield (ksi), UTS (ksi)`. The files are written with `encoding="utf-8-sig"` so Minitab reads ₀ and ² correctly. There should be no stray commas, since each comma starts a new column.
   - A4: do not leave E and yield blank it can be removed later in minitab.

## Code walkthrough (as of 2026-09-30)

Explanations of the code. Snippets are short excerpts; the notebooks have the full code.

### `stres_clalc.ipynb`

**Cell 0: setup**
```python
specimens = [f"A{i}" for i in range(1, 8)] + [f"S{i}" for i in range(1, 8)]
D = np.zeros(len(specimens))
data = {}
```
- `[f"A{i}" for i in range(1, 8)]` is a **list comprehension**: it builds a list by running the expression once for each `i`. `range(1, 8)` gives 1…7 (the stop value is excluded).
- `+` on two lists joins them, giving 14 names: `A1 … A7, S1 … S7`.
- `np.zeros(14)` creates an array of zeros ahead of time, so the loop can fill it by position with `D[i] = ...`.
- `data = {}` is an empty dict. It gets filled as `name → DataFrame`.

**Cell 1: stress loop**
- `out_dir.mkdir(exist_ok=True)` creates the output folder. `exist_ok=True` means it doesn't raise an error if the folder already exists.
- `for i, name in enumerate(specimens)`: `enumerate` gives a counter `i` alongside each item. `i` indexes `D`, and `name` picks the file.
- `float(input(...))`: `input()` always returns text, so `float()` converts it to a number. In the prompt, `name[0]` is `"A"` and `name[1:]` is `"1"`, which gives `AD1`.
- `data_dir / f"{name}.csv"`: on a `Path`, `/` joins path parts. This works on any OS.
- `pd.read_csv(...)` returns a DataFrame and takes the headers from the first row. `df["Force (lbf)"]` selects a column by its header.
- `df["Stress (psi)"] = 4 * df["Force (lbf)"] / (np.pi * D[i]**2)`: assigning to a new column name adds that column. The math is **vectorized**: it runs on every row at once, with no inner loop. `**` is the power operator.
- `df.to_csv(..., index=False)`: `index=False` stops pandas from writing the row numbers as an extra first column.
- **Saving the diameters** (after the loop):
  ```python
  diameters = pd.DataFrame({"Specimen": specimens, "d_o (in)": D})
  diameters.to_csv("diameters.csv", index=False)
  ```
  `specimens` and `D` are a list and an array in the same order, so rows pair **by position**.

**Cell 2:** `data["A1"].head()` shows the first 5 rows as a check.

### `material_properties.ipynb`

**Cell 0: load every file and plot all 14 curves in one window**
- `%matplotlib qt` is a notebook "magic" command, not Python. It must run before pyplot draws anything. To switch backends, restart the kernel.
- `sorted(data_dir.glob("*.csv"))`: `glob` finds every file matching the pattern, but in no guaranteed order. `sorted` puts them in **alphabetical** order (A1…A7, S1…S7), which matches the subplot grid.
- `file_path.stem` is the file name without the folder or the `.csv`, e.g. `A1`.
- `np.loadtxt(file_path, delimiter=",", skiprows=1)` reads the numbers into a 2-D array (rows × 5 columns). `skiprows=1` skips the header row because `loadtxt` only reads numbers. The headers are lost, so use the column-index table above.
- `table[:, 4]`: `:` means all rows and `4` means column 4 (stress). The result is a 1-D array.
- `fig, axes = plt.subplots(2, 7, figsize=(24, 8), constrained_layout=True)` creates one figure containing a 2×7 grid of axes. `constrained_layout=True` spaces them so the labels don't overlap.
- `axes.flatten()` turns the 2×7 grid into a flat list of 14, so `axes[i]` gives the i-th plot.
- `ax.plot(strain100 / 100, stress)`: x and y must be the same length. `/ 100` divides every element.
- `f"{name} Stress vs. Strain"` is an **f-string**: `{...}` inserts the value.
- `plt.show()` goes **after** the loop, so the window opens once with all plots drawn.
- A triple-quoted string inside a loop works as a comment: Python evaluates it and does nothing with it. `#` is the usual way to write a comment.

**Cell 1: UTS**
- `uts[name] = table[:, 4].max()` stores the maximum of the stress column under each specimen's key.
- `max()` vs `.max()`: Python's built-in `max()` works on a NumPy array, but it steps through the elements one at a time in Python. `table[:, 4].max()` or `np.max(table[:, 4])` does the same job with NumPy's faster built-in version. The answer is the same either way.
- `uts[name]` means "the entry in the dict `uts` whose key is the value of `name`". On the first pass `name` is `"A1"`, so it's the same as `uts["A1"]`.
  - `uts[name]` (no quotes) uses whatever string is stored in the variable `name`.
  - `uts["name"]` (with quotes) looks for a key that is literally the text `name`, which doesn't exist.

  | Where it appears | Meaning |
  |---|---|
  | Left of `=`: `uts[name] = 47100.0` | **Write:** creates the key if it's new, or replaces its value if it exists |
  | Anywhere else: `print(uts[name])` | **Read:** gets the value, or raises `KeyError` if the key isn't there |
- **Error seen:** `IndexError: invalid index to scalar variable` came from writing `uts = max(...)` without `[name]`. That replaced the whole dict with one number, so `uts[name]` then tried to index a float.

**Cell 2: Young's modulus**
- The dicts `E`, `b` and `r2` are created **before** the loop so they collect all 14 results. If they were created inside the loop, they'd be reset on every pass.
- `lo = 0.0002`, `hi = 0.002` are in in/in, i.e. 0.02% and 0.2%.
- `for name, table in data.items()`: `.items()` gives `(key, value)` pairs, so this reuses the arrays loaded in cell 0.
- **Boolean mask:** `elastic = (strain > lo) & (strain < hi)`
  - Each comparison returns an array of True/False, one per row.
  - `&` is elementwise AND. Python's `and` doesn't work on arrays.
  - The parentheses are required because `&` binds more tightly than `>`.
  - `strain[elastic]` keeps only the rows where the mask is True. Applying the **same** mask to strain and stress keeps the points paired.
- `linregress(x, y)` (from `scipy.stats`) fits a least-squares line. The result has `.slope` (E), `.intercept` (b), `.rvalue` (R² = `rvalue**2`) and `.stderr`.
- Format codes: `:.3e` is scientific notation with 3 decimals, `:.1f` is fixed-point with 1 decimal, and `:.4f` is 4 decimals.
- **A4 separate fit:** it uses a window of 0.16–0.2%, which is above the stuck 0.153% reading. `data["A4"][:,4][A4_elastic]` selects the stress column first and then applies the mask. The result is stored in `A4E` / `A4_res`.
- In the loop, `E["A4"]` keeps the value from the bad data (R² ≈ 0.08). **This is on purpose:** the report compares results before and after removing the defective set.
- Sanity check for E: A specimens are ≈ 1.0 × 10⁷ psi (aluminum) and S specimens are ≈ 3.0 × 10⁷ psi (steel).

**Cell 3: 0.2% offset line and yield strength**
- `E["A4"] = A4E` is a toggle: comment it out to use the defective value. It **overwrites** the dict entry, so every later cell that reads `E` (including the Minitab export) sees the corrected value. Re-running cell 2 puts the bad value back.
  - `b["A4"] = A4b` overwrites the intercept the same way (`A4b = A4_res.intercept`, set in cell 2).
- `for i, (name, table) in enumerate(data.items())`: the parentheses unpack each `(name, table)` pair, and `i` is the counter.
- `offset = E[name] * (strain - 0.002) + b[name]` uses the **full** strain array, not `elastic_strain`. Yield is at about 0.6%, which is outside the fit window, so a line built only from the window would stop before it crosses the curve.
- `ax.plot(strain, offset, "--", label="0.2% offset")`: `"--"` draws a dashed line. The `label=` strings are shown by `ax.legend()`.
- `ax.set_ylim(0, stress.max() * 1.1)`: the offset line is steep (about 80 ksi at 1% strain), which is above UTS. Without a y-limit, matplotlib zooms out to fit the line and squashes the curve. `* 1.1` leaves 10% headroom above UTS.
- **Optional: cut the curves at UTS.** The strain is frozen after UTS, so both lines pile up there:
  ```python
  i_uts = np.argmax(stress)
  strain = strain[:i_uts + 1]   # +1 keeps the UTS row itself
  stress = stress[:i_uts + 1]   # same slice on both keeps the pairs
  ```

**Cells 3 and 4: finding yield (where the offset line crosses the curve)**

NumPy has no single "find where two curves cross" function. The code combines a few steps: subtract the curves, find where the difference changes sign from + to −, then interpolate between the two rows on either side.
- `diff = s - off`: before yield the curve is above the offset line (`diff > 0`); after yield it's below (`diff < 0`). Yield is the first row where `diff` goes from + to −.
- `s`, `e`, `off` are `stress`, `strain`, `offset` cut to `[:i_uts + 1]`, so the search stops at UTS. After UTS the strain is frozen, so a crossing there would be meaningless.
- `yield_02 = {}` and `yield_strain = {}` are created before the loop, like `E` and `b`.
- `intersect = np.where((diff[:-1] > 0) & (diff[1:] <= 0))[0]`
  - `diff[:-1]` is rows 0 … n−2 and `diff[1:]` is rows 1 … n−1: the same array shifted by one row. Comparing them elementwise checks each row against the next, with no inner loop.
  - `&` combines the two True/False arrays: row k is + **and** row k+1 is − or exactly 0.
  - `<= 0` (not `< 0`) also catches a point that lands exactly on the line.
  - `np.where(mask)` returns a tuple with one array of indices per dimension. The data is 1-D, so `[0]` takes the only one.
- **No crossing:** `if len(intersect) == 0` stores `np.nan` and prints a message instead of crashing.
- **Interpolation:** `k = intersect[0]` is the first crossing. The crossing is somewhere between rows k and k+1:
  ```python
  t = diff[k] / (diff[k] - diff[k + 1])    # fraction 0..1 of the way from row k to k+1
  yield_02[name]     = s[k] + t * (s[k + 1] - s[k])
  yield_strain[name] = e[k] + t * (e[k + 1] - e[k])
  ```
  - With ~1000 rows the points are close together, so `s[k]` alone would be nearly the same. `t` estimates the exact crossing.
  - Same result: `np.interp(0, [diff[k+1], diff[k]], [s[k+1], s[k]])`. The x-values given to `np.interp` must be increasing, which is why they're listed in reverse.
- `ax.plot(yield_strain[name], yield_02[name], "ro", label="0.2% yield")`: `"ro"` is a red circle marker. `ax = axes[i]` is set at the top of the loop so the dot lands on the right panel.
- **Checks:**
  - Early rows: at small strain the offset line is very negative (about −0.002·E ≈ −20 ksi for aluminum), so `diff` is clearly positive. The negative-force rows and negative starting strain don't create a false crossing. Confirm the red dot sits at the knee of every curve.
  - A4: if its line never crossed, `yield_02["A4"]` would be `NaN`, and `to_csv` writes that as an empty cell. The plan says not to leave it blank, so check A4's printed result.
- **Results (2026-09-30, measured diameters, corrected A4 E):** A specimens ≈ 42.0–45.6 ksi and S specimens ≈ 123.6–128.5 ksi, all at ε ≈ 0.006. A4 is the outlier at ≈ 58.9 ksi, ε ≈ 0.0079, which matches its high UTS (see Data notes).

**Cell 4: yield strength, 14 windows**

A copy of cell 3 with three changes. The yield search and the A4 toggle are identical.
- **One window vs. 14 windows:** `plt.subplots` runs once per figure you want.
  - Cell 3 (**one window**, 14 panels): `fig, axes = plt.subplots(2, 7, ...)` and `axes = axes.flatten()` go **before** the loop, and `ax = axes[i]` picks the panel.
  - Cell 4 (**14 windows**): `fig, ax = plt.subplots(figsize=(6, 4))` goes **inside** the loop, so each pass makes a new figure. With no grid arguments, `ax` is a single axes object.
- The loop is `for name, table in data.items():` with no `enumerate`. There's no `axes[i]` to pick, so the counter `i` isn't needed.
- `plt.show()` is still after the loop, so all 14 windows open at once.
- Each figure is saved to `plots/` with `fig.savefig(...)` (see **Saving plots to files** below).

**Saving plots to files**

Every figure has a `.savefig()` method that writes it to an image file. There are two ways to use it.

*Option A (used in cell 4): save inside the loop that draws the plots*
```python
plot_dir = Path("plots")
plot_dir.mkdir(exist_ok=True)          # before the loop, same idea as out_dir in stres_clalc

for name, table in data.items():
    fig, ax = plt.subplots(figsize=(6, 4))
    # ... plotting code ...
    fig.savefig(plot_dir / f"{name}_yield.png", dpi=200, bbox_inches="tight")   # last line inside the loop
plt.show()
```
- Cell 3 has only one figure, so it would be saved once **after** the loop: `fig.savefig(plot_dir / "all_yield.png", dpi=200, bbox_inches="tight")`.

*Option B: a separate cell that saves every open window*

Run it after cells 0, 3 and/or 4, while their windows are still open:
```python
plot_dir = Path("plots")
plot_dir.mkdir(exist_ok=True)

for num in plt.get_fignums():          # ID numbers of every open figure
    fig = plt.figure(num)              # get the figure with that number
    fig.savefig(plot_dir / f"figure_{num}.png", dpi=200, bbox_inches="tight")
```
- Figures only have numbers unless you name them. Pass `num=` when creating each one: `plt.subplots(figsize=(6, 4), num=f"{name} yield")`. The string becomes the window title and the figure's label, so the save line can use `f"{fig.get_label()}.png"` as the file name.

| Part | Meaning |
|---|---|
| `plot_dir / f"{name}_yield.png"` | Joins folder and file name with `/`, like `data_dir / f"{name}.csv"` |
| `.png` | The extension sets the format. `.pdf` or `.svg` are vector graphics: sharp at any zoom, good for the report. |
| `dpi=200` | Resolution. The default (~100) looks blurry in a report; 200–300 is sharper. |
| `bbox_inches="tight"` | Trims extra white border so titles and labels aren't cut off |

- Put `savefig` **before** `plt.show()`. With `qt` either order works, but with other backends `show()` can leave an empty figure and the saved image comes out blank.
- To save without opening windows, add `plt.close(fig)` right after `savefig` in the loop.
- Re-running overwrites files with the same name without asking.
- `plots/` is kept separate from `data_with_stress/`, so image files don't mix with the specimen CSVs that cell 0 loads.

**Cell 5:** `plt.close("all")` closes every plot window, including cell 0's. With `%matplotlib qt`, windows stay open until closed. `plt.close(fig)` closes one figure.

**Cell 6: results table and Minitab export**
- **A4 R² toggle:** `r2["A4"] = A4_res.rvalue**2` at the top. Cell 3/4's toggle overwrites `E["A4"]` and `b["A4"]` but not `r2`, so without this line A4's row would pair the corrected E with the bad R² (0.08). To export A4's defective values, comment out the toggle lines in cell 3 (or 4) **and** this one.
- **Diameters:**
  ```python
  diam = pd.read_csv("diameters.csv", index_col="Specimen")
  d_o = diam["d_o (in)"]
  A_o = np.pi * d_o**2 / 4
  ```
  - `index_col="Specimen"` makes that column the row labels (A1 … S7), so `d_o` lines up by name with the dicts.
  - `diam["d_o (in)"]` is a **Series**: a 1-D column of values with labels. `A_o` is calculated for all 14 at once.
  - If `diameters.csv` doesn't exist yet, this raises `FileNotFoundError`: run `stres_clalc.ipynb` first.
- **Building the DataFrame:**
  - **From arrays or lists** (like `diameters` in `stres_clalc`): `pd.DataFrame({"Specimen": names, "UTS (psi)": uts_array})`. Each key becomes a header and each value becomes a column. All columns must be the same length, and rows are matched **by position**.
  - **From dicts or Series keyed by specimen** (like here): rows are matched **by key**, and the keys become the index (`A1 … S7`). A key that's missing from one dict becomes `NaN`.
  - `pd.Series(E) / 1e6` turns the dict into a Series so the whole column can be divided at once (psi → Msi). A plain dict can't be divided. Yield and UTS use `/ 1e3` (psi → ksi).
- **Rounding:** `results.round({"d₀ (in)": 4, "E (Msi)": 2, ...})`. Each number is the **decimal places** kept after the decimal point.
  - The dict maps column header → decimals. Columns not listed are left alone. `results.round(2)` rounds every numeric column to 2.
  - `.round()` returns a **new** DataFrame, so assign it back with `results = ...`, or the rounding is lost.
  - Negative decimals round left of the point: `round(-2)` turns 46891 into 46900.
  - Exact halves round to the nearest **even** digit: 0.125 → 0.12, 0.135 → 0.14.
  - The CSV gets the rounded values, but the dicts (`E`, `yield_02` …) keep full precision. Pick decimals to match how precise the measurement is (e.g. 4 for calipers read to 0.0005 in).
- `results.index.name = "Specimen"` then `results = results.reset_index()` turns the index into a normal `Specimen` column.
- **Splitting A and S:** `results[results["Specimen"].str.startswith("A")]`. `.str.startswith("A")` gives a True/False mask, one per row, and `results[mask]` keeps the True rows, like a NumPy boolean mask.
- `results_A.to_csv("results_A.csv", index=False)`: `NaN` is written as an empty cell, so check A4 before exporting.

## Checking the data as you go

| What to check | Command | What to expect |
|---|---|---|
| Folder the notebook runs in | `Path.cwd()` | `.../tensile_test`. Relative paths only work from here. |
| Folder exists / full path | `data_dir.exists()`, `data_dir.resolve()` | `True`, absolute path |
| Files found | `sorted(data_dir.glob("*.csv"))`, `len(...)` | 14 files, A1 … S7 |
| Parts of a path | `file_path.name`, `.stem`, `.suffix`, `.parent` | `A1.csv`, `A1`, `.csv`, folder |
| Array size | `table.shape` | `(rows, 5)`, about 1000 rows |
| Dimensions / type | `table.ndim`, `table.dtype` | `2`, `float64`; a column like `table[:, 2]` has `ndim` 1 |
| Look at rows | `table[:5]`, `table[-5:]` | first / last 5 rows |
| Value range | `strain.min()`, `strain.max()`, `stress.max()` | strain in in/in should be about 0.01 at most |
| Diameters loaded | `diam`, `d_o["A1"]` | 14 rows, about 0.5 in |
| Row of UTS | `np.argmax(stress)` | index where the flat strain tail starts |
| Points in the fit window | `elastic.sum()` | number of True rows (True counts as 1). Use it when tuning `lo`/`hi`. |
| Arrays still paired | `elastic_strain[name].shape == elastic_stress[name].shape` | `True` |
| Missing values | `np.isnan(x).any()` | `False` |
| Dict contents | `data.keys()`, `len(data)`, `E["A1"]`, `list(E.items())` | 14 keys |
| Fit quality | `r2[name]`, `res.stderr` | R² ≈ 0.999 (A4 in the loop ≈ 0.08) |
| DataFrame | `df.head()`, `df.shape`, `df.columns`, `df.dtypes`, `df.describe()` | numeric columns are `float64`, not `object` |
| What is this? | `type(x)` | `dict`, `numpy.ndarray`, `DataFrame`, `Path` … |
