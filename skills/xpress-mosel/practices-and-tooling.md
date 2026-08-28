# Mosel Practices & Tooling Reference

Sections 4-12 of the Mosel development standards (see `SKILL.md` for Sections 1-3 and the Review Checklist).

## 4. Data Sources & Handling

- Assignments (`:=`) cannot be used inside `initializations` blocks - only `from`/`to` data-transfer entries (plain names or `as`/`evaluation of` clauses) are valid there.
- Parameterize **all** data source names (and optionally table/range names).
- Use `assert` for data consistency checks; add `options keepassert` to apply in non-debug mode:
  ```
  options keepassert
  assert(A.size>0 and (or(i in R) A(i)<>0), "No data for 'A'")
  ```
- Prefer type `text` over `string` for any non-constant generated strings.
- Check file existence with `getfstat` before accessing.
- Use `setparam("readcnt", true)` and `getreadcnt` / `nbread` to validate data reads.
- For SQL: check `SQLsuccess` and report `SQLrowcnt` after queries.
- For mmsystem routines: check `getsysstat` after every call.

---

## 5. Error Handling

### Error Redirection
- Use `fopen("tee:mylog.txt&", F_OUTPUT)` to tee output to a log file + screen.
- Use `fopen("null:", F_ERROR)` to suppress error messages during controlled I/O.
- Always `fclose` after redirecting.

### Error Messages & Verbosity
- Use an integer `appErr` + text `appErrMsg` to track application error state.
- Use `break N` or loop labels (`break 'L2'`) to exit nested loops on error; use `return` to exit subroutines.
- Mark all user-facing messages for translation using `writeln_`, `_("...")`, `formattext(_("..."), ...)`.
- Implement verbosity levels:
  - 0: no output
  - 1: errors only
  - 2: errors + warnings *(recommended default for production)*
  - 3: errors, warnings, info
  - 4-10: increasingly detailed debug output

### I/O Error Pattern
```
setparam("ioctrl", true)
fopen("null:", F_ERROR)
initializations from DATAFILE
  ...
end-initializations
fclose(F_ERROR)
setparam("ioctrl", false)
if getparam("iostatus")<>0:
  setioerr(_("Specific I/O error message for this location"))
```

---

## 6. Debugging & Profiling

- Enable stack trace dump during development: `mosel exe -sdm 10 -g mymodel.mos`
- Use `getmodprop` (mmjobs) to retrieve model name, version, memory use, compile date.
- Use `versionnum`/`versionstr` to check Mosel or module versions at runtime.
- Use `memoryuse` to inspect memory consumption per entity or module.
- **Run the Mosel Profiler** at least once with representative data: `mosel profile mymodel.mos`

---

## 7. Solver Tuning

- Collect all solver parameter settings in one place (at model start, or just before the solver call, or in a dedicated subroutine).
- To output a matrix for tuning: use `loadprob` + `writeprob` with option `'x'` or `'p'`.
- For dynamic parameter settings, use `setdsoparam` or `command` (the latter only for development, not deployment).

---

## 8. Message Translation Markup

Mark up **all** user-facing messages for potential translation:
```
writeln_("Some message")
MSGSUBRSUCC:=_("Subroutine executed correctly")
appErrMsg:=formattext(_("Error at index (%d,%d)"), i, j)
```
Generate a message catalogue with: `mosel comp -x mymodel.mos` -> produces `mymodel.pot`.

---

## 9. Xpress Insight-Specific Rules

When the model targets Xpress Insight deployment:
- Only basic types (`boolean`, `integer`, `real`, `string`) and MP types (`mpvar`, `linctr`) with `set`/`array` structures are visible to Insight; other types must be model-internal only.
- Declare input/result data **globally** and **`public`**.
- Data input and derived calculations must not be mixed; place logic after `insightpopulate` in `INSIGHT_MODE_RUN`.
- Copy intermediate or subproblem results into supported types for Insight reporting.
- Compile for production **without** debug flags.
- Index sets of array entities managed by Insight cannot be reset - avoid designs that require clearing/reinitializing such index sets at runtime.

---

## 10. Good Modeling Practice & Performance

### Constants and Parameters
- Values that never change should be declared as **constants** (using `=` in a `declarations` block, not `:=`). Mosel handles them more efficiently:
  ```
  declarations
    NT = 3
    MAXP = 8.4
    Filename = "mydata.dat"
  end-declarations
  ```
- If a value is fixed within a run but may change between runs, use `parameters` (scalars only; no arrays or sets):
  ```
  parameters
    NT = 3
    MAXP = 8.4
    Filename = "mydata.dat"
  end-parameters
  ```
  Override at run time: `mosel exec mymodel NT=5 MAXP=7.5 Filename="newdata.dat"`

### Named Index Sets
- Always name index sets in constants or declarations; avoid repeating inline ranges like `1..N`:
  ```
  declarations
    RI = 1..N
    x: array(RI) of mpvar
  end-declarations
  sum(i in RI) x(i) >= 10
  ```
  Exception: if an index set would serve only a single array and that array may later need to be reset or have entries deleted, do not give the index set its own name - resetting/deleting the array's entries would leave the set unchanged if it was named.
- Declare the index set before the array declaration that uses it.
- If a filtered set is used in multiple loops, pre-compute it once, or group the loops into a single `forall ... do` block:
  ```
  ! Pre-compute once:
  ODD := union(i in RI | isodd(i)) {i}
  forall(i in ODD) x(i) is_integer
  forall(i in ODD) x(i) <= 5

  ! Or group into one loop:
  forall(i in RI | isodd(i)) do
    x(i) is_integer
    x(i) <= 5
  end-do
  ```
- Nested `forall` loops where each level carries its own filter condition should usually be merged into a single `forall` over the combined indices, with the conditions joined by `and`, rather than left nested (optionally with an `if` buried inside the innermost loop). A merged loop is shorter, avoids re-entering inner loops once per outer iteration, and - when the indices touch `exists()`-optimizable arrays - is generally required for the compiler to apply the sparse-loop optimization rules below across all the indices at once instead of just the innermost pair:
  ```
  ! Nested, with cond1(i,k) re-tested on every (j,m) for every (i,k):
  forall(i in I, k in K | cond1(i,k))
    forall(j in J, m in M | cond2(j,m))
      if cond3(i,k,j,m) then
        ...
      end-if

  ! Merged into a single filtered loop:
  forall(i in I, k in K, j in J, m in M | cond1(i,k) and cond2(j,m) and cond3(i,k,j,m))
    ...
  ```
- Do not use an `if` to filter inside a loop body - even in a single, non-nested loop. Whenever an `if`'s condition does not depend on anything computed inside the loop body itself (i.e. it could equally be evaluated before entering the loop), move it into the `forall(...|...)` clause as a condition on the loop indices instead:
  ```
  ! if inside the loop body:
  forall(i in I, j in J) do
    if cond(i,j) then
      ...
    end-if
  end-do

  ! condition on the loop indices instead:
  forall(i in I, j in J | cond(i,j))
    ...
  ```
  This is what makes the "merge nested loops" step above possible, and is required for the compiler's `exists()`-based sparse-loop optimizations (below) to take effect.

### Finalization of Sets and Dynamic Arrays
Mosel converts non-fixed (dynamic) sets to static ones via finalization, giving faster array access and enabling range checking.

- **Auto-finalization** (default): `initializations` blocks finalize the sets they initialize, and also the index sets of initialized dense arrays.
- Recommended sequence for best memory use and speed:
  1. Declare data arrays and sets to be initialized from external sources.
  2. Perform `initializations from`.
  3. Call `finalize(S)` explicitly if needed.
  4. Declare remaining arrays (including decision variable arrays).
- To disable auto-finalization **locally** (prefer this option over disabling globally):
  ```
  setparam("autofinal", false)
  initializations from "datafile.dat"
    ...
  end-initializations
  setparam("autofinal", true)
  ```
- To disable **globally**: add `options noautofinal` at the top of the model.
- Declare truly sparse arrays as `dynamic` or `hashmap`; do not use standard (dense) declarations for sparse data.

### Index Ordering for Sparse Arrays
For sparse array efficiency, loop index order should match the array's declaration order:
```
x: array(A, B, C) of mpvar      ! declared in order A, B, C

sum(a in A, b in B, c in C) x(a,b,c)   ! fast - matches declaration order
sum(b in B, c in C, a in A) x(a,b,c)   ! slower - declare as array(B,C,A) instead
```

### Efficient Sparse Loops with `exists`
The compiler can optimize sparse loops automatically. Rules for fast `exists`-based loops:
1. Arrays must be indexed by **named sets** (not inline ranges):
   ```
   A: dynamic array(I, J) of real             ! can be optimized
   B: dynamic array(1..1000, 1..500) of real   ! cannot be optimized
   ```
2. Use the **same named sets** in the loop as in the array declaration.
3. **Loop index order** must match the declaration order (critical for `dynamic`; `hashmap` is more flexible).
4. `exists()` calls must appear **first** in the condition:
   ```
   forall(i in I, j in J | exists(A(i,j)) and i+j<>10)   ! fast
   forall(i in I, j in J | i+j<>10 and exists(A(i,j)))   ! slow
   ```
5. Use `exists()` with **`and`**, not `or`:
   ```
   forall(i in I, j in J | exists(A(i,j)) and i+j<>10)   ! fast
   forall(i in I, j in J | exists(A(i,j)) or  i+j<>10)   ! slow
   ```

The two loops below produce identical output, but the first performs 1,000,000 empty-string checks (one per combination of `i1`/`i3`) while the second - which satisfies rules 1-4 above - performs only 2 (one per existing element):
```
declarations
  I1, I2, I3: range
  MyArray: dynamic array(I1, I2, I3) of string
end-declarations
I1:=1..1000; I2:=1..1000; I3:=1..1000
MyArray(2,3,4) := "Hello "
MyArray(10,3,30) := "World"
i2:=3
forall(i1 in I1, i3 in I3 | MyArray(i1,i2,i3)<>"") writeln(MyArray(i1,i2,i3))                       ! slow: 1M checks
forall(i1 in I1, i3 in I3 | exists(MyArray(i1,i2,i3)) and MyArray(i1,i2,i3)<>"") writeln(MyArray(i1,i2,i3))   ! fast: 2 checks
```
This is why an apparently redundant `exists()` check is often deliberate and performance-critical - do not remove it during cleanup. When the loop iterates over sets/lists other than the array's own declared index sets, an `exists()` test may genuinely be redundant if another condition already guarantees the cell exists - check case by case rather than assuming.

### Computational Complexity & Efficiency
- Aim to limit the computational complexity of the algorithms you write (avoid unnecessary nested loops over large sets, redundant recomputation, etc.).
- Aim to write code that is time- and memory-efficient. For example, avoid repeating a computation inside a loop when it can be done once outside it:
  ```
  mean:= sum(i in S) A(i)/S.size      ! less efficient: divides on every iteration (S.size divisions)
  mean:= (sum(i in S) A(i))/S.size    ! more efficient: sums first, divides once
  ```
- Enumerate sparse data via named-set `exists()` loops (see "Efficient Sparse Loops with `exists`" above) rather than looping over a dense index range and testing membership on every iteration.
- Each access to an element of a non-static (`dynamic`/`hashmap`) array re-does a lookup; a static (dense, finalized) array does not have this cost. When a block reads or writes the same element of a non-static array more than once, cache it locally with `with` instead of re-accessing the array:
  ```
  ! Repeated lookups on a dynamic/hashmap array element:
  if A(i,j) > 0 and A(i,j) < 10 then
    total += A(i,j)
  end-if

  ! Cache the value locally - looked up once:
  with a = A(i,j) do
    if a > 0 and a < 10 then
      total += a
    end-if
  end-do
  ```

### Compiling for Production
- Do not compile production code with debug or trace options (e.g. `-g`, `-G`); they add compile/runtime overhead and are intended for development only.
- Reserve `mosel debug`, `mosel profile`, and `mosel coverage` runs for development; deploy using `mosel comp`/`mosel exec` without those flags.

### Constraint Names and Optimizer Memory
- Naming constraints (and variables), and loading those names into the Optimizer, consumes extra memory in the solver. Only name a constraint/variable if the name is actually needed - e.g. to modify or query it later (`getact`, `setcoeff`, ...), to produce a named LP/MPS export, or because debug settings are enabled - otherwise leave it anonymous.
- Anonymous constraints are cheaper but cannot be modified or queried later (see "Constraint Operations" in `SKILL.md`) - this is the trade-off against naming.
- Leave `setparam('XPRS_LOADNAMES', ...)` (see "Viewing and Exporting the Problem Matrix" in `SKILL.md`) at its default (`false`) in production; only set it `true` when names are required or while debugging.
- Inspect the Mosel model's memory consumption with `memoryuse` before the solver call, and free/reset any large data structures that are no longer needed at that point. Only invoke `memoryuse` during development or in debug mode, not in production runs.

### Scope: Prefer Local Declarations
- Declare objects inside the subroutine that uses them (a local `declarations` block) rather than at global scope, unless the object must be shared across subroutines or (for Insight) published. Local objects are released when the subroutine returns, reducing peak memory use.

### Reusing Temporary Data Structures
- Delete, reset, or reuse auxiliary/temporary arrays, sets, and lists once they are no longer needed, rather than letting a new one accumulate on every loop iteration or subroutine call.
- Use `delcell` to remove individual entries from a `dynamic`/`hashmap` array or set, or reassign the whole object to release it in bulk.
- Where a temporary structure is used repeatedly (e.g. once per loop iteration), prefer clearing and reusing the same object over declaring/creating a new one each time.

### Recommended Model Section Order
Build the model in this sequence:
1. Constant data: declare and initialize
2. All non-constant objects: declare
3. Variable data: initialize / read / calculate
4. Decision variables: create, specify bounds
5. Constraints: declare and specify
6. Objective: declare, specify, then call `minimize`/`maximize`

### Additional Style Points
- Use `options noimplicit` to force all objects to be explicitly declared; the compiler will flag undeclared identifiers.
- Include `min` or `max` in every objective function name (e.g., `MinCost`, `MaxRevenue`), not a generic name like `OBJ`.
- Use verbs for decision variable names to emphasize that variables represent "what to do" decisions.
- Keep subroutine length to roughly one screen page; use `include` or packages to split large model files.
- Subroutines called very frequently (e.g., at every B&B node) that are performance-critical can be moved to a user module via the Mosel Native Interface.

---

## 11. Command Line & Tooling

### Standard Commands
```
mosel exec mymodel.mos     ! Compile + load + run (.mos source)
mosel mymodel              ! Same; accepts .mos or .bim
mosel comp mymodel.mos     ! Compile only; output: mymodel.bim
mosel run  mymodel.bim     ! Load and run a precompiled BIM file
mosel debug mymodel.mos    ! Start interactive debugger
mosel profile mymodel.mos  ! Profiler run; output: mymodel.mos.prof
mosel coverage mymodel.mos ! Code coverage run
mosel lslib                ! List available modules and packages
mosel exam -ps mmxprs      ! Show parameters and subroutines of mmxprs
mosel exam -a  mybim.bim   ! Show annotations of a model or package
mosel exam -h              ! Display Mosel version info and search paths
mosel -V                   ! Show Mosel version string
```

Compile to a specific output path:
```
mosel comp mymodel.mos -o mydir/mybim.bim
```

### Setting Runtime Parameters from the Command Line
```
mosel exec mymodel NT=5 DATAFILE="mydata.dat"
mosel run  mymodel NT=5 DATAFILE="mydata.dat"
mosel      mymodel NT=5 DATAFILE="mydata.dat"
```

### Debugger Commands
| Category | Commands |
|---|---|
| Breakpoints | `break delete bcond breakpoints breaksub` |
| Execution | `cont next step finish model` |
| Output | `display undisplay list print info exportprob` |
| Listing | `lsattr lslibs lslocal lsmods lssymb` |
| Stack | `up down where` |
| Options | `option` |
| Quit | `quit` |

Example debugging session:
```
mosel debug mymodel.mos
break 20                         ! Set breakpoint at line 20
cont                             ! Run to breakpoint
print D                          ! Print value of symbol D
info Arr                         ! Show info about Arr (e.g. array size)
lsmods                           ! Show model info (memory usage etc.)
quit
```

Debugging across submodels:
```
mosel debug main.mos
breaksub 1                       ! Stop at the start of each submodel
cont
break 25 sub.mos                 ! Set breakpoint in the submodel
display SNumbers                 ! Watch on object SNumbers
bcond 2-2 SNumbers.size < 10    ! Conditional breakpoint
cont
quit
```

---

## 12. Character Encoding

Mosel 4.0 and later use UTF-8 internally. Models and data files that contain only standard ASCII characters (code points 0 to 127) are unaffected.

### Non-ASCII Model Source Files
If you edit a model file with an editor that saves in a non-UTF-8 encoding (e.g., CP1252 on Windows), declare the encoding with an annotation at the very top of the source file:
```
!@encoding CP1252
model "my testmodel"
  ...
```

### Non-ASCII Data Files
For text data files that are not UTF-8, apply the `enc:` prefix when opening or accessing them:
```
! Open a data file in GB18030 encoding:
fopen("enc:GB18030,testdata.txt", F_INPUT)

! Copy and re-encode a CSV file, adding a UTF-8 BOM:
fcopy("myfile.csv", F_INPUT, "enc:UTF-8+bom,mynewfile.csv", F_OUTPUT)
```

Inside `initializations` blocks:
```
initializations to "mmsheet.csv:enc:sys,output.csv"
  ...
end-initializations
```
The alias `sys` selects the default system encoding (matching the behavior of Mosel versions before 4.0).
Other aliases: `raw sys wchar fname tty ttyin stdin stdout stderr`

### Checking and Converting Encodings
Check which encoding is configured on the current system:
```
xprnls info
```

Convert a file between two encodings:
```
xprnls conv -f CP1252 -t UTF8 -o outfile.txt myfile.txt
```

List all available xprnls commands:
```
xprnls
```

The XPRNLS library (for use from C/API code) handles conversions between UTF-8 and local encodings; it is platform-independent with no external dependencies. It implements UTF-8/16/32 (LE+BE), ISO-8859-1/15, ASCII, and CP1252 natively; other encodings depend on the operating system.

Note: Xpress Workbench always uses UTF-8, independent of the system encoding setting.
