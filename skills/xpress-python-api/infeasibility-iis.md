# Debugging Infeasible and Unbounded Problems

## Debugging Infeasible Problems (IIS)

When a problem is infeasible, use **IIS (Irreducible Infeasible Set)** to find the minimal subset of constraints that conflict.

```python
from xpress.enums import MIPStatus, LPStatus

p.optimize()

if p.attributes.mipstatus == MIPStatus.INFEAS:
    print("Problem is infeasible - analyzing IIS...")

    # Find the first IIS (Xpress 9.8+)
    p.firstIIS(0)  # 0 = default flags

    # Get IIS data
    row, col, rtype, btype, duals, rdcs, isrows, icols = p.getIISData(0)

    print(f"IIS contains {len(row)} constraints and {len(col)} variable bounds")

    # Get names of conflicting constraints
    if len(row) > 0:
        print("\nConflicting constraints:")
        for r, dual in zip(row, duals):
            row_name = p.getNameList(xp.Namespaces.ROW, r, r)[0]
            print(f"  - {row_name} (dual: {dual:.4f})")

    # Get names of conflicting variable bounds
    if len(col) > 0:
        print("\nConflicting variable bounds:")
        for c, bt, rdc in zip(col, btype, rdcs):
            col_name = p.getNameList(xp.Namespaces.COLUMN, c, c)[0]
            bound_type = "lower" if bt == 'L' else "upper"
            print(f"  - {col_name} {bound_type} bound (rc: {rdc:.4f})")
```

**Understanding IIS results:**
- **Rows in IIS**: Constraints that together cause infeasibility
- **Columns in IIS**: Variable bounds involved in the conflict
- **Dual values**: Indicate how much the objective would improve if relaxed

**Finding multiple IIS:**
```python
if p.nextIIS() == 0:
    row2, col2, *_ = p.getIISData(2)
else:
    print("No more IIS found")
```

**Common infeasibility patterns:**
1. Conflicting bounds: `x >= 10` and `x <= 5`
2. Over-constrained demand: Sum of supplies < total demand
3. Mutually exclusive constraints: Binary variables forced to incompatible values

## Debugging Unbounded Problems (Primal/Dual Rays)

When a problem is unbounded, use **primal rays** to identify which variables can grow indefinitely.

**IMPORTANT:** Presolve must be disabled to retrieve primal rays.

```python
# CRITICAL: Disable presolve before solving
p.controls.presolve = 0

p.optimize()

if p.attributes.lpstatus == xp.LPStatus.UNBOUNDED:
    print(f"LP Status: {p.attributes.lpstatus.name}")

    # Verify ray is in original (non-presolved) space
    if p.attributes.presolvestate & xp.PresolveState.PROBLEMLPPRESOLVED:
        print("Error: Primal ray is in presolved space")
    else:
        ray = p.getPrimalRay()

        if ray is not None:
            ncols = p.attributes.cols
            obj_coeffs = p.getObj(0, ncols - 1)

            print("Variables contributing to unboundedness:")
            for i in range(ncols):
                if abs(ray[i]) > 1e-6:
                    var = p.getVariable(i)
                    print(f"  {var.name}: ray={ray[i]:.6f}, obj_coeff={obj_coeffs[i]:.4f}")
```

**Understanding primal ray results:**
- **Non-zero ray values**: Variables that can be scaled in that direction
- **Ray direction**: How much each variable increases relative to others

When a subproblem is infeasible, use dual rays to build feasibility cuts:

```python
p.controls.presolve = 0   # Required -- presolve may not produce a ray
p.optimize()

if p.attributes.lpstatus == xp.LPStatus.INFEAS:
    dray = p.getDualRay()  # Returns None if ray not available (9.9+)
    if dray is None:
        # Fall back to alternative (e.g. no-good cut)
        pass
    else:
        # Use dray to build feasibility cut
        pass
```

**Primal rays for iterative constraint generation (robust optimization):**

```python
import numpy as np

# HIST is a sparse matrix (N products x M scenarios)
# use_vars: list of xpress variables, one per product
while p.attributes.lpstatus == xp.LPStatus.UNBOUNDED:
    ray = p.getPrimalRay()
    if ray is None:
        break

    for s in range(M):
        dot_product = (np.array(ray) @ HIST[:, s]).item()
        if dot_product > 0:
            p.addConstraint(xp.Dot(use_vars, HIST[:, s]) <= RHS)
            break

    p.optimize()
```

**Comparison: IIS vs Primal Rays:**

| Aspect | Infeasible (IIS) | Unbounded (Primal Ray) |
|--------|------------------|------------------------|
| Function | `firstIIS()`, `getIISData()` | `getPrimalRay()` |
| Presolve | Works with presolve | **Must disable presolve** |
| Problem types | LP, MIP | LP, convex QP only |

**Common unboundedness causes:**
1. Missing upper bounds on variables with positive objective coefficients
2. Missing constraints for resource usage
3. Constraint coefficients accidentally set to zero

## IISOPS Bit-Vector (protect restrictions from IIS deletion)

```python
# These are bit flags - combine with bitwise OR
xp.IISOps.BINARY     # 1  - keep binary integralities
xp.IISOps.ZEROLOWER  # 2  - keep zero lower bounds
xp.IISOps.FIXEDVAR   # 4  - keep fixed variables
xp.IISOps.BOUND      # 8  - keep all variable bounds
# Example: protect bounds and binary integralities
p.controls.iisops = xp.IISOps.BOUND | xp.IISOps.BINARY
```
