---
name: xpress-python-api
description: Xpress Python API modeling guide - quick reference for variables, constraints, objectives, indicators, SOS, piecewise linear, callbacks, IIS/infeasibility and unbounded-problem debugging, low-level matrix-based problem loading, and Pandas/NumPy/SciPy integration.
when_to_use: Activate when the user asks about creating or debugging Xpress optimization models in Python, Xpress Python API syntax, callbacks (including Benders/cut injection), IIS or infeasible/unbounded debugging (primal/dual rays), low-level matrix/CSC problem loading (loadLP/addRows/addCols), Pandas/NumPy/SciPy integration with xp.Dot, performance patterns, or common Python API errors and gotchas.
---

# Xpress Python Modeling Guide

**Version:** Xpress 9.9 (2026) | **Python Support:** 3.10-3.14

Quick reference and modeling patterns for the FICO Xpress Python API (`import xpress as xp`).

## Documentation Location

- **Full Python API reference:** `python-interface.md` is the complete Xpress Python API reference -- every method, its full signature, arguments, and return values -- generated from the Xpress XML doc sources so it always matches the installed solver version. It ships alongside the Xpress install at `%XPRESSDIR%\docs\solver\MD\python-interface.md` (Windows) or `$XPRESSDIR/docs/solver/MD/python-interface.md` (Linux/Mac); also available on the Xpress docs website. The full set of Markdown docs (generated from the XML sources) is also downloadable as a zip from the Xpress docs website.
- **Example notebooks:** [github.com/fico-xpress/python-notebooks](https://github.com/fico-xpress/python-notebooks) (public)
- **Python examples (.py files):** `<xpressmp-install>/examples/python` (standard install path: `C:\xpressmp\examples\python`)

### API Lookup Rule -- prefer the MD file over `help()`

**Prefer `python-interface.md` over `python -c "import xpress as xp; help(xp.problem.someMethod)"` to check signatures.**

Both are generated from the same XML doc source, so neither is more accurate
than the other -- the MD file is just faster to search (a single `grep`)
than spawning Python and paging through `help()` output:

```bash
# Search for a method signature
grep -A 20 "problem.repairWeightedInfeas" "path/to/python-interface.md"
```

If `python-interface.md` cannot be found at any of the paths above, tell the
user it's missing and ask them to point you to their `XPRESSDIR` install path
or the Xpress docs website to download the Markdown docs zip. `help()` is a
reasonable fallback in that case since it ships with the installed package --
do not fall back to guessing signatures from memory.

## Reference files (read on demand)

- **[callbacks.md](./callbacks.md)** -- full callback API: message/lplog/
  miplog/intsol/preintsol/newnode/prenode/optnode/nodelpsolve/gapnotify
  signatures, the callback summary table, Benders decomposition via
  `preintsol` cut injection, and 9.9 callback thread-safety rules. Read when
  the user asks about callbacks, monitoring/controlling B&B, lazy
  constraints, cut injection, Benders decomposition, or "why does my callback
  raise an exception."
- **[infeasibility-iis.md](./infeasibility-iis.md)** -- diagnosing
  INFEASIBLE and UNBOUNDED problems: IIS (`firstIIS`/`getIISData`, multiple
  IIS), primal rays (`getPrimalRay`) for unboundedness, dual rays
  (`getDualRay`) for feasibility cuts, the IISOPS bit-vector, and an
  IIS-vs-ray comparison table. Read when the user says "my model is
  infeasible," "how do I debug an infeasible model," "unbounded," "primal
  ray," "dual ray," or asks about `mipstatus == INFEAS` / `lpstatus ==
  UNBOUNDED`.
- **[low-level-api.md](./low-level-api.md)** -- matrix-based problem
  construction: `loadLP`/`loadMIP`/`loadQP`/`loadMIQP`, `addRows`/`addCols`/
  `addNames`, low-level IIS functions.
- **[pandas-numpy.md](./pandas-numpy.md)** --
  `dtype='xpressobj'` DataFrame variables, vectorized modeling with
  groupby/boolean indexing, Pandas 3.0 compatibility, `xpress.ndarray` matrix
  operations, SciPy sparse support in `xp.Dot()`, and the xp.Dot
  symbolic-expression performance deep-dive (why pre-computing with NumPy
  avoids a 100-200x slowdown). Read when the user asks about Pandas/NumPy/
  SciPy integration, "why is my xp.Dot slow," vectorizing model construction,
  or DataFrame-based variables.

## Quick Reference - Modern API (Xpress 9.9)

---
**WARNING: TOP 3 MOST COMMON MISTAKES - ALWAYS CHECK FOR THESE!**

1. **Use `xp.SolStatus` (not `xp.SolveStatus`) for solution checking!**
   - `xp.SolveStatus` is a different enum (for `SOLVESTATUS` attribute) -- not what you want
   - WRONG: `if p.attributes.solstatus == xp.SolveStatus.OPTIMAL:` -> wrong enum type
   - CORRECT: `if p.attributes.solstatus == xp.SolStatus.OPTIMAL:` -> use xp.SolStatus
   - See "solstatus vs solvestatus" in Common Gotchas for when to use each

2. **`solstatus`/`lpstatus`/`mipstatus` are already enum objects - use `.name` directly!**
   - WRONG: `xp.SolStatus(p.attributes.solstatus).name`  -> Redundant wrapping
   - WRONG: `xp.LPStatus(p.attributes.lpstatus).name`  -> Redundant wrapping
   - CORRECT: `p.attributes.solstatus.name`  -> Directly returns "OPTIMAL", "INFEASIBLE", etc.
   - CORRECT: `p.attributes.lpstatus.name`  -> Directly returns "OPTIMAL", "UNBOUNDED", etc.

3. **ALWAYS use enum constants -- NEVER magic numbers for any status or type check!**
   - WRONG: `if p.attributes.solstatus == 1:` -> unreadable, fragile
   - WRONG: `if p.attributes.iissolstatus == 2:` -> unreadable
   - CORRECT: `if p.attributes.solstatus == xp.SolStatus.OPTIMAL:`
   - CORRECT: `if p.attributes.iissolstatus == xp.IISSolStatus.COMPLETED:`
   - This applies to ALL enum types -- see "Enum Constants Reference" section below.

See "Common Gotchas" section for more details.

---

### Creating a Problem

```python
import xpress as xp

p = xp.problem(name="my_model")
```

### Variables

```python
# Continuous variable (default)
x = p.addVariable(lb=0, ub=100, name="x")

# Binary variable
y = p.addVariable(vartype=xp.binary, name="y")

# Integer variable
z = p.addVariable(vartype=xp.integer, lb=0, ub=10, name="z")

# Multiple variables - returns Xpress array (for use with xp.Dot)
x = p.addVariables(10, name="x")           # 10 variables: x[0]..x[9]
x = p.addVariables(4, 5, name="x")         # 4x5 matrix
x = p.addVariables(3, 4, 5, name="x")      # 3x4x5 tensor

# Binary array
y = p.addVariables(10, vartype=xp.binary, name="y")
```

### Constraints

```python
# Basic linear constraints
p.addConstraint(x + y <= 10)
p.addConstraint(2*x - 3*y >= 5)
p.addConstraint(x + y == 7)

# Using xp.Sum for summations
p.addConstraint(xp.Sum(x[i] for i in range(10)) <= 100)

# Multiple constraints at once (generator)
p.addConstraint(x[i] + y[i] <= 10 for i in range(5))
```

#### Naming Constraints (9.9+)

`addConstraint()` now accepts a `name=` argument. Names appear in LP files, solver logs, and IIS output -- making debugging much easier.

```python
# Form 1: single constraint with a plain name
p.addConstraint(x + y <= 10, name="capacity")

# Form 2: list of constraints with a list of names (one-to-one)
p.addConstraint([c1, c2, c3], name=["lower_bound", "upper_bound", "budget"])

# Form 3: any collection with a prefix -- solver appends (0), (1), ...
# NOTE: use a list [...] not a generator (...) so the cell output shows
#       constraint names instead of "<generator object ...>"
p.addConstraint([x[i] + y[i] <= 10 for i in range(5)], name="capacity")

# Read names back to decode IIS or dual output
row_names = p.getNameList(xp.Namespaces.ROW)          # all rows
row_names = p.getNameList(xp.Namespaces.ROW, 0, 2)    # rows 0-2 (first/last optional in 9.9)

# Use case: make IIS output human-readable
p.firstIIS(0)
miisrow, *_ = p.getIISData(1)
row_names = p.getNameList(xp.Namespaces.ROW)
print("IIS constraints:", [row_names[r] for r in miisrow])

# Use case: label shadow prices by constraint name
names = p.getNameList(xp.Namespaces.ROW)
duals = p.getDuals()
for name, dual in zip(names, duals):
    print(f"  {name}: {dual:.4f}")
```

### Vectorized Operations with xp.Dot

**Use `xp.Dot` for better performance** when multiplying arrays of coefficients with arrays of variables.

```python
# Use np.array for plain numeric data -- xp.array is only needed for arrays
# that hold Xpress objects (like variables)
costs = np.array([1.0, 2.0, 3.0, 4.0, 5.0])
x = p.addVariables(5, name="x")  # Returns Xpress array

# Efficient dot product - avoids the Python-level loop below
p.setObjective(xp.Dot(costs, x))

# Equivalent but slower -- xp.Sum itself is fast, the Python loop building
# up individual terms one at a time is what costs time for large n:
# p.setObjective(xp.Sum(costs[i] * x[i] for i in range(5)))

# Matrix-vector operations
A = np.array([[1, 2], [3, 4], [5, 6]])  # 3x2 matrix
b = np.array([10, 20, 30])              # RHS values, one per row
x = p.addVariables(2, name="x")
# xp.Dot(A, x) <= b creates all 3 constraints at once (no Python loop needed)
p.addConstraint(xp.Dot(A, x) <= b)
```

**When to use xp.Dot:**
- Large number of terms (100+)
- Repeated similar expressions
- Matrix/vector operations
- Performance-critical model building

**Performance tip:** pre-compute coefficient matrices with NumPy before
involving variables in `xp.Dot()` -- intermediate symbolic expressions can be
100-200x slower. See
**[pandas-numpy.md](./pandas-numpy.md)**.

### Objective Function

```python
# Minimize (default)
p.setObjective(3*x + 2*y)
p.setObjective(expr, sense=xp.ObjSense.MINIMIZE)

# Maximize
p.setObjective(3*x + 2*y, sense=xp.ObjSense.MAXIMIZE)

# Quadratic objective
p.setObjective(x**2 + 2*x*y + y**2 + 3*x + 4*y)

# Using xp.Sum
p.setObjective(xp.Sum(cost[i] * x[i] for i in range(n)))

# Using xp.Dot (faster for large models)
costs = np.array([...])
x = p.addVariables(n, name="x")
p.setObjective(xp.Dot(costs, x))
```

### Indicator Constraints

```python
# When binary y == 1, enforce x <= 10
p.addIndicator(y == 1, x <= 10)

# When binary y == 0, enforce x <= 0 (common for "if closed, no flow")
p.addIndicator(y == 0, x <= 0)

# General form: p.addIndicator(condition, implied_constraint)
```

### SOS Constraints (Special Ordered Sets)

```python
# SOS1: At most one variable in the set can be non-zero
p.addSOS([x[0], x[1], x[2]], [1, 2, 3], type=1)

# SOS2: At most two consecutive variables can be non-zero
p.addSOS(variables_list, weights_list, type=2)

# With name (name is a keyword argument)
sos = p.addSOS([y[0], y[1]], [1, 2], type=1, name="my_sos")
```

### Piecewise Linear Functions

Two APIs available -- prefer `xp.pwl()` when slopes are known, `addPwlCons()` when only coordinates are known.

**`xp.pwl()` -- interval dict with linear expressions (slope-based)**

```python
import xpress as xp
import numpy as np

p = xp.problem()
x = p.addVariable(ub=4)

# Continuous concave PWL (breakpoints at 0,1,2,3,4)
pw = xp.pwl({(0, 1):      10*x,
             (1, 2): 10 +  3*(x-1),
             (2, 3): 13 +  2*(x-2),
             (3, 4): 15 +    (x-3)})

# Approximate sin(freq*x): use breakpoints + slopes as linear expression per interval
N, freq = 10, 5
step = (2 / math.pi) / (N - 1)
bp = np.array([i * step for i in range(N)])    # x breakpoints
v  = np.sin(freq * bp)                         # function values
s  = freq * np.cos(freq * bp)                  # derivatives (slopes)

pw2 = xp.pwl({(bp[i], bp[i+1]):
              v[i] + s[i] * (x - bp[i]) for i in range(N - 1)})

p.setObjective(pw2, xp.ObjSense.MAXIMIZE)
p.optimize()

print(p.getSolution(x))
print(xp.evaluate(pw2, problem=p))  # evaluate PWL value at solution
```

**`addPwlCons()` -- coordinate-based (x/y breakpoint arrays)**

```python
p = xp.problem()
x  = p.addVariable()
pw = p.addVariable()   # resultant variable (output of PWL function)

N, freq = 10, 5
bp, v = create_segments(N, freq)   # returns x-breakpoints and y-values

# addPwlCons(col, res, starts, x_vals, y_vals)
# col: input variables, res: output variables,
# starts: starting index per function in x_vals/y_vals
p.addPwlCons([x], [pw], [0], bp, v)

p.setObjective(pw, xp.ObjSense.MAXIMIZE)
p.optimize()
```

**Key differences:**
- `xp.pwl()`: takes dict of `{(x_start, x_end): linear_expr}` -- one variable only per PWL; supports discontinuous functions via separate segments
- `addPwlCons()`: takes flat arrays of all breakpoint coordinates; better for batch-adding multiple PWL constraints
- `xp.pwl()` can appear directly in `setObjective()`/`addConstraint()`; `addPwlCons()` requires a
  resultant variable (`pw` above) which is then used in `setObjective()`/`addConstraint()` instead
- `xp.evaluate([pw1, pw2], problem=p)` retrieves PWL values at solution

### Solving

```python
# Basic solve
p.optimize()

# CRITICAL: Use SolStatus (not SolveStatus!) for checking solution status
from xpress.enums import SolStatus

# CORRECT: Check if solution is feasible or optimal
if p.attributes.solstatus in [SolStatus.FEASIBLE, SolStatus.OPTIMAL]:
    print("Solution found")

# Use solvestatus to find out why the solve stopped (e.g. hit the time limit)
from xpress.enums import SolveStatus
if p.attributes.solvestatus == SolveStatus.STOPPED:
    print("Solve stopped early")

# Get objective value
obj = p.attributes.objval

# Get solution values
x_val = p.getSolution(x)           # Single variable
vals = p.getSolution([x, y, z])    # Multiple variables
vals = p.getSolution()             # All variables
```

### Debugging Infeasible / Unbounded Problems (IIS, Primal/Dual Rays)

When `mipstatus`/`lpstatus` comes back INFEAS or UNBOUNDED, read
**[infeasibility-iis.md](./infeasibility-iis.md)** for the full IIS
(`firstIIS`/`getIISData`) and primal/dual ray (`getPrimalRay`/`getDualRay`)
workflows, the IISOPS bit-vector, and the IIS-vs-ray comparison table.

### Controls (Parameters)

```python
# Single control - use p.controls attribute
p.controls.timelimit = 300         # Time limit in seconds
p.controls.miprelstop = 0.01       # 1% MIP gap tolerance
p.controls.outputlog = 1           # Enable output

# Multiple controls (3+) - use dictionary
p.setControl({
    'timelimit': 300,
    'miprelstop': 0.01,
    'outputlog': 1,
    'threads': 4,
    'presolve': 1
})

# Get control value
val = p.controls.timelimit

# Common controls:
# - timelimit: time limit (seconds)
# - miprelstop: relative MIP gap tolerance
# - mipabsstop: absolute MIP gap tolerance
# - threads: number of threads
# - outputlog: 0=silent, 1=normal
# - presolve: 0=off, 1=on (default)
# - heuremphasis: heuristic effort (0-5)
# - cutstrategy: cut generation strategy (-1 to 3)
```

### Callbacks

Xpress supports callbacks (`add*Callback`) for monitoring and controlling
optimization: message, lplog, miplog, intsol, preintsol, newnode, prenode,
optnode, nodelpsolve, gapnotify. For signatures, the callback table,
Benders/cut-injection patterns via `preintsol`, and the 9.9 callback
thread-safety rules, read **[callbacks.md](./callbacks.md)**.

### Reading/Writing Models

```python
# Write to file
p.writeProb("model.lp")        # LP format
p.writeProb("model.mps")       # MPS format
p.writeProb("model", "l")      # LP format (explicit flag)
# No flag is needed for MPS -- it's the default output format.

# Read from file
p.readProb("model.lp")
p.readProb("model.mps")
```

## Low-Level API (Matrix-Based Problem Loading)

For tools/libraries that build problems directly from matrices:
`loadLP`/`loadMIP`/`loadQP`/`loadMIQP`, `addRows`/`addCols`/`addNames`,
low-level IIS functions all live in
**[low-level-api.md](./low-level-api.md)**.

## Common Patterns

### Facility Location with Indicator Constraints

```python
import xpress as xp

p = xp.problem()

# Binary: is facility i open?
y = p.addVariables(n_facilities, vartype=xp.binary, name="open")

# Continuous: flow from facility i to customer j -- ub broadcasts capacity[i] across each row
x = p.addVariables(n_facilities, n_customers, lb=0, ub=capacity, name="flow")

# Demand constraints -- sum over facilities (axis 0) for each customer
p.addConstraint(x.sum(axis=0) >= demand)

# Indicator: if facility closed, no flow (per-pair, so this stays a loop)
for i in range(n_facilities):
    for j in range(n_customers):
        p.addIndicator(y[i] == 0, x[i][j] <= 0)

# Objective: minimize fixed + transport costs
p.setObjective(
    xp.Dot(fixed_cost, y) + (transport_cost * x).sum()
)

p.optimize()
```

## Common Gotchas

**See also the CRITICAL MISTAKES banner at the top of this file for the top 4 issues.**

### 1. `c.name` and `v.name` are always populated

Xpress always assigns a name to every constraint and variable. If the user did not provide one, the system assigns a default (e.g. `"R1"`, `"C1"`). **Never guard with `if c.name`** -- it is always truthy.

```python
# WRONG -- the else branch can never trigger
label = c.name if c.name else f"R{row_idx}"

# CORRECT -- c.name is always set
label = c.name
```

### 2. Message callback can receive None

```python
def callback(prob, msg, *args):
    if msg is not None:  # Always check!
        print(msg)
```

### 3. SOS weights must match variables length

```python
p.addSOS([x[0], x[1], x[2]], [1, 2, 3])  # 3 weights for 3 vars
```

### 4. Use np.array for numbers, xp.array only for Xpress objects

```python
costs = np.array([1.0, 2.0, 3.0])  # Plain numbers -- np.array is fine (even better)
x = p.addVariables(3, name="x")    # Use addVariables, not list comprehension
p.setObjective(xp.Dot(costs, x))
```

### 5. addVariable / addVariables: duplicate names raise SolverError

Assigning the same `name` to two variables raises an error at creation time:

```python
x = p.addVariable(name="x")
y = p.addVariable(name="x")   # SolverError: Duplicate column names are not allowed
```

The same applies to `addVariables` -- if you call it twice with the same base name, the auto-generated names (`x[0]`, `x[1]`, ...) collide on the second call:

```python
a = p.addVariables(3, name="x")  # x[0], x[1], x[2]
b = p.addVariables(3, name="x")  # SolverError -- x[0] already exists
```

For `addVariables`, omitting `name=` does **not** avoid this: the default base name is still `x`, so two omitted-name calls collide just like the example above. Use a distinct name prefix on each call instead.

### 6. Pandas dtype='xpressobj' doesn't work with xp.Dot

```python
# For xp.Dot: use p.addVariables directly
x = p.addVariables(n, vartype=xp.binary)
p.setObjective(xp.Dot(costs, x))

# For DataFrame operations: use dtype='xpressobj'
df['selected'] = pd.Series(p.addVariables(n, vartype=xp.binary), dtype='xpressobj')
p.setObjective((df['cost'] * df['selected']).sum())  # Pandas ops work
```

## Attributes Reference

### Problem Attributes (read after solve)

```python
p.attributes.objval          # Objective value
p.attributes.mipstatus       # MIP status
p.attributes.solstatus       # Solution status
p.attributes.rows            # Number of rows
p.attributes.cols            # Number of columns
p.attributes.mipents         # Number of integer entities
p.attributes.qelems          # Number of quadratic elements
p.attributes.bestbound       # Best bound (dual bound)
p.objcontrols                # Objective controls (9.9+, also available in callbacks)
```

## Enum Constants Reference

**RULE: Always use enum constants. Never use magic numbers for any status or type check.**

All enums are available directly on the `xp` module (e.g. `xp.SolStatus.OPTIMAL`) or via `from xpress.enums import SolStatus`.

### Most Commonly Used Enums

```python
# Solution status (check after optimize())
xp.SolStatus.NOTFOUND       # 0 - no solution found
xp.SolStatus.OPTIMAL        # 1 - optimal solution
xp.SolStatus.FEASIBLE       # 2 - feasible but not proven optimal
xp.SolStatus.INFEASIBLE     # 3 - infeasible
# Usage: if p.attributes.solstatus == xp.SolStatus.OPTIMAL:

# Solve status (reason solve stopped)
xp.SolveStatus.UNSTARTED    # 0
xp.SolveStatus.STOPPED      # 1 - hit a limit (time, node, etc.)
xp.SolveStatus.FAILED       # 2
xp.SolveStatus.COMPLETED    # 3 - solved to completion
# Usage: if p.attributes.solvestatus == xp.SolveStatus.COMPLETED:

# LP status
xp.LPStatus.UNSTARTED       # 0
xp.LPStatus.OPTIMAL         # 1
xp.LPStatus.INFEAS          # 2
xp.LPStatus.CUTOFF          # 3
xp.LPStatus.UNFINISHED      # 4
xp.LPStatus.UNBOUNDED       # 5
xp.LPStatus.CUTOFF_IN_DUAL  # 6
xp.LPStatus.UNSOLVED        # 7
xp.LPStatus.NONCONVEX       # 8
# Usage: if p.attributes.lpstatus == xp.LPStatus.OPTIMAL:

# MIP status
xp.MIPStatus.NOT_LOADED     # 0
xp.MIPStatus.LP_NOT_OPTIMAL # 1
xp.MIPStatus.LP_OPTIMAL     # 2
xp.MIPStatus.NO_SOL_FOUND   # 3
xp.MIPStatus.SOLUTION       # 4
xp.MIPStatus.INFEAS         # 5
xp.MIPStatus.OPTIMAL        # 6
xp.MIPStatus.UNBOUNDED      # 7
# Usage: if p.attributes.mipstatus == xp.MIPStatus.OPTIMAL:

# IIS computation status
xp.IISSolStatus.UNSTARTED   # 0 - not started or license/input error
xp.IISSolStatus.FEASIBLE    # 1 - no IIS found (problem is feasible)
xp.IISSolStatus.COMPLETED   # 2 - IIS computation completed normally
xp.IISSolStatus.UNFINISHED  # 3 - interrupted, partial IIS may be available
# Usage: if p.attributes.iissolstatus == xp.IISSolStatus.COMPLETED:

# Stop reason
xp.StopType.NONE            # 0 - not stopped
xp.StopType.TIMELIMIT       # 1
xp.StopType.CTRLC           # 2
xp.StopType.NODELIMIT       # 3
xp.StopType.ITERLIMIT       # 4
xp.StopType.MIPGAP          # 5
xp.StopType.SOLLIMIT        # 6

# Basis status (for LP warm-starting)
xp.BasisStatus.NONBASIC_LOWER  # 0
xp.BasisStatus.BASIC           # 1
xp.BasisStatus.NONBASIC_UPPER  # 2
xp.BasisStatus.SUPERBASIC      # 3

# Objective sense -- always use ObjSense enum, not xp.minimize/xp.maximize
xp.ObjSense.MINIMIZE        # 1
xp.ObjSense.MAXIMIZE        # -1
# Usage:
p.setObjective(expr, sense=xp.ObjSense.MINIMIZE)
p.setObjective(expr, sense=xp.ObjSense.MAXIMIZE)

# Solution available
xp.SolAvailable.NOTFOUND    # 0
xp.SolAvailable.OPTIMAL     # 1
xp.SolAvailable.FEASIBLE    # 2
```

The IISOPS bit-vector (protecting bounds/integralities from IIS deletion) is
covered in **[infeasibility-iis.md](./infeasibility-iis.md)**.

## Integration with NumPy/Pandas/SciPy

For `dtype='xpressobj'` DataFrame variables, vectorized modeling idioms
(groupby/boolean indexing), Pandas 3.0 compatibility, `xpress.ndarray`, SciPy
sparse support in `xp.Dot()`, and the xp.Dot symbolic-expression performance
deep-dive, read
**[pandas-numpy.md](./pandas-numpy.md)**.

## When to Use This Skill

Use this skill when the user asks about:
- Creating optimization models in Python with Xpress
- Xpress Python API syntax (addVariable, addConstraint, etc.)
- Indicator constraints, SOS constraints in Python
- Quadratic objectives in Python
- Callbacks in Python (addMessageCallback, addLplogCallback, addMiplogCallback, addIntsolCallback, addPreIntsolCallback, addPrenodeCallback, addOptnodeCallback) -- see [callbacks.md](./callbacks.md)
- Debugging infeasible/unbounded problems (IIS, primal/dual rays) -- see [infeasibility-iis.md](./infeasibility-iis.md)
- Low-level matrix-based problem loading (loadLP/addRows/addCols) -- see [low-level-api.md](./low-level-api.md)
- Pandas/NumPy/SciPy integration -- see [pandas-numpy.md](./pandas-numpy.md)
- Performance optimization with xp.Dot
- Common Python API errors and gotchas

## Important: Deprecation Check

**After modeling any new problem or extending an existing one, always verify there are no deprecation warnings in the logs.** Look for the word "Deprecated" in the output. Deprecation warnings indicate use of old API patterns that should be updated to the modern Xpress 9.8+ API.

## Checking for Version-Specific Behavior Changes

If code behaves unexpectedly after an Xpress version upgrade, or a bug is
suspected in a specific function, check the
[release notes](https://www.fico.com/fico-xpress-optimization/docs/latest/relnotes)
rather than assuming -- individual per-release bug fixes and behavior
changes are not tracked in this skill.
