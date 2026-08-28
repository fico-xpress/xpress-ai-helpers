# Callbacks

Xpress supports various callbacks for monitoring and controlling the optimization process. The modern API uses `add*Callback` naming pattern.

**Signature:** `p.add*Callback(callback_function, data, priority)`
- `callback_function`: Function to call
- `data`: User data passed to callback (can be `None`)
- `priority`: Higher = called earlier (positive integer)

```python
# Message callback - monitor solver output
def message_cb(prob, data, msg, msgtype):
    if msg is not None:
        print(msg)

p.addMessageCallback(message_cb, None, 0)

# LP log callback - called after every LPLOG iterations
def lplog_cb(prob, data):
    print(f"LP iteration, obj={prob.attributes.lpobjval}")

p.addLplogCallback(lplog_cb, None, 0)

# MIP log callback - called when MIP log is printed
def miplog_cb(prob, data):
    print(f"MIP: best={prob.attributes.mipobjval}, bound={prob.attributes.bestbound}")

p.addMiplogCallback(miplog_cb, None, 0)

# Integer solution callback - called when integer solution found (after acceptance)
def intsol_cb(prob, data):
    print(f"New integer solution: {prob.attributes.mipobjval}")

p.addIntsolCallback(intsol_cb, None, 0)

# Pre-integer solution callback - called BEFORE solution is accepted
# Can reject solutions and add cuts when soltype=0
def preintsol_cb(prob, data, soltype, cutoff):
    # soltype: 0=node relaxation, 1=heuristic, 2=user provided
    obj = prob.attributes.lpobjval
    print(f"Found solution: {obj}, soltype={soltype}")
    # Return (reject, newcutoff)
    # reject: 0 to accept, 1 to reject
    # newcutoff: new cutoff value (or None)
    return (0, None)

p.addPreIntsolCallback(preintsol_cb, None, 0)

# New node callback - called when a new B&B node is created
def newnode_cb(prob, data, parentnode, newnode, branch):
    print(f"New node {newnode} created from parent {parentnode}")

p.addNewnodeCallback(newnode_cb, None, 0)

# Prenode callback - called before LP relaxation at node is solved
# Return nonzero to declare the node infeasible
def prenode_cb(prob, data):
    # Solution not available yet
    return 0

p.addPrenodeCallback(prenode_cb, None, 0)

# Optimal node callback - called after LP solved, after cuts/heuristics
# Return nonzero to declare the node infeasible
def optnode_cb(prob, data):
    # Can access LP solution, add lazy constraints
    return 0

p.addOptnodeCallback(optnode_cb, None, 0)

# Node LP solved callback - after LP solved, before cuts/heuristics
# No return value
def nodelpsolve_cb(prob, data):
    pass  # Early access to node LP solution

p.addNodeLPSolvedCallback(nodelpsolve_cb, None, 0)
```

**Key callbacks:**
| Callback | When called | Common use |
|----------|-------------|------------|
| `addMessageCallback` | Solver output messages | Logging, progress monitoring |
| `addLplogCallback` | Every LPLOG simplex iterations | LP progress tracking |
| `addMiplogCallback` | MIP log line printed | MIP progress tracking |
| `addIntsolCallback` | Integer solution accepted | Solution logging |
| `addPreIntsolCallback` | Before solution accepted | Solution filtering, rejection, cut injection |
| `addNewnodeCallback` | New B&B node created | Node tracking, tree analysis |
| `addPrenodeCallback` | Before node LP solved | Custom preprocessing |
| `addOptnodeCallback` | After node LP + cuts/heuristics | Lazy constraints, branching |
| `addNodeLPSolvedCallback` | After node LP, before cuts | Early solution access |
| `addGapNotifyCallback` | Gap target reached | Custom termination |

## Pre-Integer Solution Callback (Benders / Cut Injection)

The `preintsol` callback is powerful for advanced algorithms like Benders decomposition.

**Critical behavior by soltype:**

| soltype | Description | Can add cuts? | Typical action |
|---------|-------------|---------------|----------------|
| 0 | Node relaxation | YES | Add cuts via `addCuts()` |
| 1 | Heuristic solution | NO | Reject if invalid |
| 2 | User-provided solution | NO | Reject if invalid |

**Adding cuts when soltype=0:**
```python
if soltype == 0:
    p.addCuts([ncuts], [sense], [rhs], [start], indices, coefficients)
    return (0, None)  # Return 0 to KEEP cuts!
```

**The return value: `(ifreject, new_cutoff)`**
- `return (0, None)` - Accept solution (any cuts added are kept)
- `return (1, None)` - Reject solution (any cuts added are dropped; for soltype=0 the node is also dropped)
- Second element optionally updates the MIP cutoff; pass `None` to leave unchanged

**Performance pattern:**

**WARNING:** Rejecting heuristic solutions (`soltype != 0`) discards potentially good incumbents found by MIP heuristics, and the solver still pays the full cost of finding them. Xpress does not support adding cuts outside a B&B node, so if you always end up rejecting non-zero soltypes here, prefer disabling heuristics via `HEURSTRATEGY`/`HEUREMPHASIS = 0` instead -- rejecting solutions is only worth doing if you have a specific reason to distrust individual heuristic solutions (e.g., they may violate problem-specific constraints not modeled in Xpress).

```python
def preintsol_callback(p, data, soltype, cutoff):
    xhat = p.getCallbackSolution(x_vars)

    # NOTE: if soltype != 0 are heuristic/user solutions -- only reject
    # them if they genuinely cannot be valid (e.g. custom feasibility check).
    # Blindly rejecting non-zero soltypes discards good MIP heuristic solutions.
    if soltype != 0:
        return (1, None)

    # soltype == 0: Safe to solve expensive subproblem
    subproblem_result = expensive_validation(xhat)

    if not subproblem_result.is_valid:
        p.addCuts([1], ['L'], [rhs], [0, len(indices)], indices, coefficients)
        return (0, None)
    return (0, None)
```

**Full Benders decomposition example:** a complete, runnable Benders
decomposition using `preintsol` for cut injection ships with Xpress at
`%XPRESSDIR%\examples\python\modeling_examples\benders_decomp.py` (Windows)
or `$XPRESSDIR/examples/python/modeling_examples/benders_decomp.py`
(Linux/Mac) -- standard Windows install path:
`C:\xpressmp\examples\python\modeling_examples\`.
It relies on a subproblem-modeling technique (a dummy linking variable/
constraint per first-stage variable) to turn plain LP duals into valid cut
coefficients -- read the full file rather than a partial snippet here, since
that technique is what makes the duals-as-coefficients pattern correct.

**When presolve must be disabled for callbacks:** presolve can remove or
merge variables/constraints, which shifts the column/row indices your
callback code relies on (e.g. indices passed to `addCuts()`). This is not a
blanket rule for every callback -- only disable presolve if your callback
logic depends on original-space indices staying stable, as in the Benders
example above:

```python
p.setControl({
    'presolve': 0,
    'mippresolve': 0,
    'symmetry': 0
})
```

## Callback Thread Safety (9.9)

Inside any callback, accessing variable/constraint object properties is subject to these rules:

- **Safe:** `var.name`, `constraint.name`, `sos.name` -- always return user-defined names
- **Raises exception:** `var.lb`, `var.ub`, `constraint.rhs`, and other data properties -- access these **before** the solve and store in a local variable
- **Removed (9.9):** `var.getCallbackSolution()`, `var.getCallbackRedCosts()`, `constraint.getCallbackSlacks()`, `constraint.getCallbackDuals()` -- use `callback_problem.getCallbackSolution(var)` etc. instead

```python
# WRONG (9.9): accessing var.lb inside callback raises exception
def cb(prob, data, ...):
    limit = my_var.lb   # RuntimeError!

# CORRECT: copy data before the solve
var_lb = my_var.lb      # read before p.optimize()
def cb(prob, data, ...):
    limit = var_lb      # safe
```

**Remove callbacks:**
```python
p.removeMessageCallback(message_cb, data)  # Remove specific callback
p.removeMessageCallback()                   # Remove all message callbacks
```
