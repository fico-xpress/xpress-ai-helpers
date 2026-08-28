# Low-Level API (Matrix-Based Problem Loading)

For tools/libraries that build problems directly from matrices, Xpress provides low-level functions. These were significantly updated in Xpress 9.8.

## Problem Loading Functions (Xpress 9.8+)

Type-specific loading functions replace the generic `loadproblem()`:

```python
# LP: loadLP() - Linear Programming (column-major / CSC format)
p.loadLP(
    probname="",   # Problem name string (optional)
    rowtype=None,  # Row types ('L'<=, 'G'>=, 'E'=, 'R'range, 'N' nonbinding)
    rhs=None,      # Right-hand side values
    rng=None,      # Range values (for 'R' rows; lower bound = rhs - abs(rng))
    objcoef=None,  # Objective coefficients (None = all zeros)
    start=None,    # Column start offsets into rowind/rowcoef (CSC format)
    collen=None,   # Number of nonzeros per column (optional if start has trailing entry)
    rowind=None,   # Row indices for nonzero elements
    rowcoef=None,  # Nonzero element values
    lb=None,       # Lower bounds (None = all zeros)
    ub=None        # Upper bounds (None = all infinity)
)
# Use addNames() afterward to set column/row names (see "Adding Names Separately" below)

# MIP: loadMIP() - loadLP's args plus discrete-entity/SOS args
p.loadMIP(probname="", rowtype=None, rhs=None, rng=None, objcoef=None,
          start=None, collen=None, rowind=None, rowcoef=None, lb=None, ub=None,
          coltype=None,   # Entity type chars per entry in entind: 'B'/'I'/'P'/'S'/'R'
          entind=None,    # Column indices the coltype entries apply to (sparse -- not one per column)
          limit=None,     # Partial-integer / semi-continuous limits, parallel to entind
          settype=None,   # SOS set types ('1' or '2') per set
          setstart=None,  # Start offsets into setind/refval per set
          setind=None,    # Column indices per SOS set
          refval=None)    # Reference weights per SOS set member

# QP: loadQP() - Quadratic Programming (LP signature + quadratic terms)
p.loadQP(
    probname="", rowtype=None, rhs=None, rng=None, objcoef=None,
    start=None, collen=None, rowind=None, rowcoef=None, lb=None, ub=None,
    objqcol1=None,  # First column indices for Q matrix
    objqcol2=None,  # Second column indices for Q matrix
    objqcoef=None   # Coefficients for Q matrix
)

# MIQP: loadMIQP() - loadQP's args plus the same discrete-entity/SOS args as loadMIP
p.loadMIQP(probname="", rowtype=None, rhs=None, rng=None, objcoef=None,
           start=None, collen=None, rowind=None, rowcoef=None, lb=None, ub=None,
           objqcol1=None, objqcol2=None, objqcoef=None,
           coltype=None, entind=None, limit=None,
           settype=None, setstart=None, setind=None, refval=None)
```

## Adding Names Separately

In Xpress 9.8+, names can be added separately using `addNames()` with `xp.Namespaces`. In 9.9, `first` and `last` are optional (default to all objects of that type):

```python
import xpress as xp

# Add column (variable) names
p.addNames(xp.Namespaces.COLUMN, colnames, 0, n_cols - 1)

# Add row (constraint) names
p.addNames(xp.Namespaces.ROW, rownames, 0, n_rows - 1)

# Get names back -- first/last optional in 9.9
col_names = p.getNameList(xp.Namespaces.COLUMN)              # all columns
row_names = p.getNameList(xp.Namespaces.ROW)                 # all rows
row_names = p.getNameList(xp.Namespaces.ROW, start_idx, end_idx)  # range
```

## Adding Rows and Columns

```python
# Add rows (constraints) - Xpress 9.8+
p.addRows(rowtype, rhs, rng=None, start=None, colind=None, rowcoef=None)

# Add columns (variables) - Xpress 9.8+
p.addCols(objcoef=None, start=None, rowind=None, rowcoef=None, lb=None, ub=None)
# No names/coltypes args -- use addNames() (see above) and chgColType() (below) separately

# Change column types (for integer/binary variables)
p.chgColType(indices, coltypes)  # coltypes: 'C', 'B', 'I'
```

## IIS Functions (Low-Level)

```python
p.firstIIS(flags)                    # Find first IIS (flags: 0 = default)
p.nextIIS()                          # Find next IIS
row, col, rtype, btype, duals, rdcs, isrows, icols = p.getIISData(iis_index)
row_names = [p.getNameList(xp.Namespaces.ROW, r, r)[0] for r in row]
```
