---
name: xpress-mosel
description: Use this skill when writing, debugging, reviewing, improving, or deploying Mosel model code (`.mos` files), Mosel packages, or any FICO Xpress Optimization model. Triggers on requests like "write a Mosel model", "review this Mosel code", "check my .mos file", "create a Mosel package", "prepare Mosel code for publication", "productize a Mosel model", or when the user pastes Mosel code for feedback.
---

# Mosel Model Writing & Review

Apply the following guidelines from the FICO Xpress Mosel development standards when writing or reviewing Mosel code.

### Reference Documentation
The authoritative Mosel reference documentation ships with the installation at `$XPRESSDIR/docs/mosel/MD`, in particular:
- `moselquickref.md` - quick syntax reference
- `moselref.md` - full language reference
- `moselug.md` - user guide
- `mosellibs.md` - C library reference

Consult these files directly for anything not covered here or when in doubt about exact syntax/behavior.

### Additional Skill Reference Files
This skill's guidance continues in two additional files, loaded on demand:
- `practices-and-tooling.md` - Sections 4-12: Data Sources & Handling, Error Handling, Debugging & Profiling, Solver Tuning, Message Translation Markup, Xpress Insight-Specific Rules, Good Modeling Practice & Performance, Command Line & Tooling, Character Encoding.
- `abbreviations.md` - the full Standard Identifier Abbreviations table (see Section 3, "Naming Conventions", below).

## 1. Language Quick Reference

### Model File Structure
```
model "model_name"
  options noimplicit, explterm    ! Compiler options
  version M.m.r                  ! Version number
  uses "mmxprs", "mmodbc"        ! Modules (compiler directives)

  parameters
    PARAM = default_value        ! Runtime-overridable scalars only
  end-parameters

  declarations
    ...
  end-declarations

  ! Body: initializations, subroutines, constraints, solver calls

end-model
```
A line break acts as an expression terminator; to continue an expression across lines, end the line with an operator that implies continuation (e.g., `+ - , *`).

### Data Structures
| Type | Description |
|---|---|
| `array(idx) of T` | Dense indexed collection; add `dynamic` or `hashmap` for sparse |
| `set of T` | Unordered unique elements; `range` = contiguous integer set |
| `list of T` | Ordered, duplicates allowed |
| `record ... end-record` | Named fields of any types |
| `name = existing_type` | User-defined type alias (in declarations block) |
| `T1 or T2` / `any` | Union type (holds one of several types at runtime) |

```
! Array examples
A: array(1..5) of real
B: array(range, set of string) of integer
C: dynamic array(I, J) of real    ! sparse

! Set / range
S: set of string
R: range

! List
L: list of integer

! Record
ARC: array(ARCSET:range) of record
  Source, Sink: string
  Cost: real
end-record

! Union
u: string or real
a: any
```

Array indices generally do not need to be set/declared explicitly - they are populated automatically as the array is populated (via `initializations`, direct assignment, or `create`). Bulk assignment to an array uses `::`:
```
RET:: (1..3)[1, 2, 3]           ! Bulk-assigns array RET from a literal list
```

### Selection Statements
```
if cond then
  ...
elif cond then
  ...
else
  ...
end-if

if c=1: writeln('c equals 1')     ! Single-statement shorthand

case val of
  1, 2: writeln("1 or 2")
  3..6: do
    writeln("3 to 6")
  end-do
else
  writeln("other")
end-case
```

### Loop Constructs
```
forall(i in S | cond) stmt
forall(i in S) do ... end-do

with f='F1', t=1 do ... end-do   ! forall stopped after first iteration

while (cond) do ... end-do

repeat
  ...
until cond

break        ! Exit current loop
break n      ! Exit n nested loops
break 'L'    ! Exit to labeled loop L
next         ! Next iteration of current loop
next 'L'     ! Next iteration of labeled loop L

! Labeled loops:
'L1': repeat
  'L2': while (cond1) do
    if cond2 then break 'L1' end-if
  end-do
until cond3

! Counter variable in bounded loop:
cnt := 0.0
writeln((sum(cnt as counter, i in S | cond) expr) / cnt)
```
Loop indices (e.g. `i` in `forall(i in S)`) must not be pre-declared; they are implicitly declared by the loop construct itself and scoped to it.

### Operators
| Category | Operators / Syntax |
|---|---|
| Arithmetic | `+ - * / ^ mod div` |
| Aggregate | `sum(i in S) ...`  `prod(i in S) ...`  `min(i in S) ...`  `max(i in S) ...`  `count(i in S | cond)` |
| Logical | `and or not` ; `and(i in S) ...` ; `or(i in S) ...` |
| Comparison | `< > = <> <= >=` |
| Set | union `+`, intersect `*`, difference `-` ; `union(i in S) ...` ; `inter(i in S) ...` |
| Set membership | `elt in S` ; `elt not in S` ; subset `S1 <= S2` ; superset `S1 >= S2` |
| List | concat `+` or `sum` or `union` ; truncate `-` |
| Assignment | `:= += -=` |
| Reference | `->proc` (reference to subroutine) |

For `linctr` objects: `:=` drops the constraint type (<=/>=/=); `+=` and `-=` retain it.

### Decision Variable Types
```
x: mpvar                      ! Continuous, 0 to +infinity by default
x <= 10                       ! Upper bound
y(2) is_free                  ! Free variable (unbounded below)
b is_binary                   ! Binary (0/1)
d is_integer                  ! General integer
d <= 25                       ! Upper bound on integer
x is_partint 10               ! Partial integer (integer up to 10, continuous beyond)
y(3) is_semcont 5             ! Semi-continuous (0 or >= 5)
```

### Constraint Operations
```
Ctr := 2*x + y <= 10          ! Named constraint
Ctr += 2*x                    ! Modify (keep type)
settype(Ctr, CT_UNB)          ! Change type
sethidden(Ctr, true)          ! Hide from solver
val := getact(Ctr)            ! Activity value
getvars(Ctr, vars)            ! Variables in constraint
Ctr := 0                      ! Delete (reset) constraint
```
Anonymous constraints (no name assigned) cannot be modified or queried later.

### Problem Handling
```
declarations
  myprob: mpproblem
end-declarations

with myprob do
  x + y >= 0
end-do
```
`mpproblem` can be combined: `mypbtyp = mpproblem and somepbtype`

### Key Built-in Functions and Procedures
**Arrays/sets:** `create exists delcell isdynamic finalize getsize getnbdim`
**Elements:** `getelt getfirst getlast findfirst findlast`
**Lists:** `gethead gettail cutelt cutfirst cutlast cuthead cuttail reverse splithead splittail`
**Math:** `ceil floor round abs exp log ln sqrt cos sin arctan isodd`
**Special reals:** `isfinite isinf isnan`
**Random:** `random setrandseed`
**Aggregate:** `minlist maxlist`
**Inline if:** `if(cond, val_true, val_false)`
**File I/O:** `fopen fclose fselect getfid getfname iseof fflush fskipline`
**Read/write:** `read readln write writeln fwrite fwriteln`
**String:** `strfmt substr`
**Constraints:** `getcoeff getcoeffs setcoeff getvars gettype settype sethidden setname setrange`
**Integrality:** `makesos1 makesos2 setmipdir`
**Solution:** `getobjval getsol getrcost getdual getact getslack getprobstat`
**Control:** `getparam setparam localsetparam restoreparam exit`
**Versioning:** `versionnum versionstr`
**Memory/debug:** `memoryuse dumpcallstack assert`
**Date/time:** `currentdate currenttime timestamp`
**Unions:** `geteltype getstruct gettypeid isdefined`
**Publish (Insight):** `publish unpublish`
**Misc:** `asproc compare datablock newmuid reset setioerr setmatherr`

### Solution Status Constants (mmxprs)
Always test `getprobstat` before reading any solution values:
```
case getprobstat of
  XPRS_OPT: writeln('optimal')
  XPRS_INF: writeln('infeasible')
  XPRS_UNB: writeln('unbounded')
  XPRS_UNF: writeln('unfinished')
else
  writeln('unexpected status')
end-case
```

### Viewing and Exporting the Problem Matrix
```
! Matrix held by solver (mmxprs) - use for solver tuning:
loadprob(ObjFn)
writeprob("out.mps", "")     ! MPS format
writeprob("out.lp",  "l")    ! LP format

! Matrix held by Mosel core:
exportprob("out", ObjFn)              ! LP format (default)
exportprob(EP_MPS, "out", ObjFn)      ! MPS format
```
Useful solver settings when examining the matrix:
```
setparam('XPRS_VERBOSE', true)
setparam('XPRS_LOADNAMES', true)
```

### Writing Solutions to Files
```
declarations
  make_sol: array(ITEMS, TIME) of real
  obj_sol: real
end-declarations

forall(i in ITEMS, t in TIME)
  make_sol(i,t) := getsol(make(i,t))
obj_sol := getobjval

initializations to 'results.dat'
  make_sol
  obj_sol
end-initializations

! Or use 'evaluation of' directly:
initializations to 'results.dat'
  evaluation of
    array(i in ITEMS, t in TIME) getsol(make(i,t)) as 'make_sol'
  evaluation of getobjval as 'obj_sol'
end-initializations
```

### Common I/O Drivers
| Driver prefix | Data source |
|---|---|
| (none / plain file path) | Mosel .dat text format |
| `mmsheet.excel:` | MS Excel (Windows only) |
| `mmsheet.xls:` / `mmsheet.xlsx:` | Generic spreadsheet (cross-platform) |
| `mmsheet.csv:` | CSV files |
| `mmodbc.odbc:` | ODBC databases |
| `mmoci.oci:` | Oracle databases |
| `mmetc.diskdata:` | mp-model style data files |

For the spreadsheet drivers (`mmsheet.xlsx:` / `mmsheet.xls:` / `mmsheet.excel:`), reading data requires either:
- named ranges defined inside the spreadsheet that correspond to the data items being imported, or
- an explicit sheet name and cell range stated per entity in the `initializations` block (not in the driver/file path), e.g.:
  ```
  initializations from "mmsheet.xlsx:mydata.xlsx"
    A as "[Sheet1$B3:D6]"
  end-initializations
  ```

For a complete list of I/O drivers and detailed spreadsheet/database examples, consult:
- `moselio.md` - whitepaper "Generalized file handling in Mosel": complete list of I/O drivers
- `moseldata.md` - whitepaper "Using ODBC and other database interfaces with Mosel": spreadsheet/database I/O examples

### Mosel Data File Format (.dat)
- Single-line comments marked with `!`
- Format: label, colon, data value(s)
- Array data enclosed in `[  ]`; values are space-separated (comma-separated format is deprecated)
- Dense format: values fill the table starting at the first position, last index varies fastest
- Sparse format: each item preceded by its index tuple in parentheses
  ```
  COST: [("Oil1" 1) 3.9  ("Oil1" 3) 4.8
         ("Oil2" 2) 7.5  ("Oil2" 3) 5.5]
  ```

### Reserved Words
Do not use any of the following as identifiers (both lower and upper case are reserved; mixed case such as `And` is not):
```
and any array as boolean break case constant count counter
declarations div do dynamic elif else end evaluation false
forall forward from function hashmap if imports in include
initialisations initializations integer inter is is_binary
is_continuous is_free is_integer is_partint is_semcont
is_semint is_sos1 is_sos2 linctr list max min mod model
mpproblem mpvar namespace next not nsgroup nssearch of
options or package parameters procedure public prod range
real record repeat requirements return set shared string
sum then to true union until uses version while with
```

### Annotations
```
!@doc.descr   Single-line annotation in category 'doc', marker 'descr'

(!@doc.       Enter category 'doc' (text on this line is ignored)
 @ descr      Value of doc.descr
 @.           Back to root category
 @mynote      Contents of '.mynote'
!)

!@mc.def descr alias doc.descr om.descr   ! Declare an alias
```
Generate model documentation:
```
mosel comp -D mymodel.mos      ! Compile with annotation data (-D flag)
moseldoc mymodel               ! Produce HTML and XML
moseldoc -o mydir -html mymodel   ! HTML only, specify output directory
moseldoc -f -xml mymodel       ! XML only, force overwrite
```

---

## 2. Model Structure

### Header (comment block)
Every model file must begin with a comment block containing:
- Name of the project
- Short description of the purpose of this file
- Optionally: build/run instructions, list of dependent files, version overview, author, creation date

### Versioning
- Always use the `version` option in all model and package files.
- Use version string format `MMM.mmm.rrr` (major.minor.release).
- Versioning rules:
  - Removed functionality -> increase **M** (major)
  - Changed/bugfix behavior -> increase **r** (release)
  - Added new functionality -> increase **m** (minor)
- Track major versions in the model header or a separate log file.

### Loading Modules and Packages
- Every used module (DSO) or package must have its own `uses` or `imports` statement - never assume transitive loading.
- DSOs always use `uses`. Main models should load application packages via `imports` (static inclusion); packages load other packages via `uses`.

### Parameters
- Every `parameters` entry must be followed by a comment explaining its purpose and possible values.
- Use parameters for all data source paths (e.g., default value `"./"`).

### Declarations
- Every declaration must be followed by a comment or `doc` annotation.
- Use logical grouping (e.g., per input/output file, or related data like demand + due date + penalty).
- If an index set is shared across tables, comment which table populates it.
- Global declarations should appear early in the model.
- When declaring an array with named index sets, the index set declarations must appear *before* the array declaration that uses them.

### Requirements Blocks (Packages)
A `requirements` block lists symbols a package requires for its processing but does not define. Unlike `declarations` blocks:
- Array entities do not need `dynamic`/`hashmap` qualifiers specified.
- Entries do not need individual comments.

### Code Structure and Formatting
- Use subroutines for data pre/postprocessing, constraint definitions (one per type/group), and output generation.
- Keep main model under 1000-1500 lines; use `include` or packages for larger models.
- Separate data handling from constraint definitions and algorithms.
- Use **spaces** (not tabs) for indentation, **2-space width**.
- Apply indentation to: the contents of `keyword ... end-keyword` blocks (e.g. `declarations`/`end-declarations`, `if`/`end-if`, `case`/`end-case`, `with`/`end-do`), the bodies of loops (`forall`/`while`/`repeat`), and continuation lines of a statement that spans several lines.
- Suggested maximum line length: **80 characters**; wrap longer statements onto continuation lines (see Operators above for valid line-break points).
- For a simple `if` with a single short "then" statement and no `else`, use the shorthand form `if <condition>: <statement>` rather than the full `if ... then ... end-if` block, unless the whole line would exceed ~100 characters:
  ```
  if c=1: writeln('c equals 1')     ! preferred for a short single statement

  if c=1 then                       ! use this form only if the shorthand line
    writeln('c equals 1')           ! would exceed ~100 characters, or an
  end-if                            ! else branch is needed
  ```
- Formatting conventions:
  ```
  ! Declarations
  myreal = real
  A: array(MySet) of real

  ! Assignments
  Ctr:= 2*x + y <= 10
  i+=1
  ratio:= x/y            ! no need to wrap fractions in parentheses

  ! Loops
  forall(i in R) do
  sum(c in Customers | FLAG(c)<>"y") VAL(c)
  while (j<10) do

  ! Conditions
  if DEBUG=1 then
  case myval of
  res:= if(flag>0, 'A', 'B')
  ```

---

## 3. Naming Conventions

### Parameters
**ALL UPPER CASE**, underscores allowed for readability:
```
MSG_MAXLOGSIZE
```

### Declarations
| Entity type | Convention | Example |
|---|---|---|
| Index set / list | First letter upper case or ALL CAPS | `Customers`, `NODES` |
| Decision variable (`mpvar`) | all lower case (possibly mixed case with first letter lower case if very long), prefer verbs | `select`, `assign` (not `x`, `y`) |
| Constraint / objective (`linctr`) | Mixed case, first letter upper | `CtrChannelCapacity`, `TotalCost` |
| Scalar immutable | ALL UPPER CASE | `INFOMESSAGE`, `ERRORNUM` |
| Scalar mutable | Mixed case or all lower case | `InfoMessage`, `cnt` |
| Data array | ALL CAPS or Mixed case (never all lower) | `DEMAND`, `CustSegment` |
| Record field | All lower case or apply conventions for the corresponding entity type | `source`, `cost` |
| Type definition | All lower case (mixed if needed to avoid ambiguity); use package/app prefix or a namespace | `svgstylesheet`, `s3objectlist`, `ins~attachment` |

### Functions and Procedures
- All **lower case**, no underscores (unless name is very long).
- If in a package, prefix with the shortened package name or apply namespacing:
  ```
  readdata
  createconstraints
  segaddcondition            ! From package "sdksegment"
  ins~getscenattach          ! From "mminsightscenario" (namespace)
  ```

### Package Public Identifiers
Always prefix with a shortened package name or define within a namespace:
```
SegCustomers: set of string    ! From "sdksegment"
procedure svgaddgroup          ! From "mmsvg"
procedure ins~clearerror       ! From "mminsightscenario" (namespace)
```

### Standard Identifier Abbreviations
Use standard abbreviated forms when composing multi-word entity or subroutine names - see the full table in `abbreviations.md`.

---

Sections 4-12 (Data Sources & Handling, Error Handling, Debugging & Profiling, Solver Tuning, Message Translation Markup, Xpress Insight-Specific Rules, Good Modeling Practice & Performance, Command Line & Tooling, Character Encoding) continue in `practices-and-tooling.md` - read that file before writing, reviewing, or deploying any Mosel code.

---

## Review Checklist

When reviewing Mosel code, check:
- [ ] Model header comment block present
- [ ] `version` option declared
- [ ] All `uses`/`imports` explicit (no transitive loading assumed)
- [ ] All parameters commented
- [ ] All declarations commented or annotated
- [ ] Naming conventions followed (see Section 3)
- [ ] Decision variables named meaningfully (not x/y/z), using verbs
- [ ] Objective function name includes 'min' or 'max' (not just 'OBJ')
- [ ] Data sources parameterized
- [ ] `assert` used for data consistency where appropriate
- [ ] Fixed values declared as constants (not mutable variables)
- [ ] Index sets named; repeated filtered sets pre-computed or loops grouped
- [ ] Initialization follows declare -> init -> finalize -> declare-dependents order
- [ ] Sparse loops: `exists()` with named sets, correct index order, `exists()` first in condition with `and`
- [ ] Error handling with `appErr`/`appErrMsg` pattern
- [ ] User-facing messages use `writeln_` / `_("...")`
- [ ] Verbosity level control present
- [ ] Indentation uses 2 spaces (not tabs)
- [ ] Subroutines used for distinct concerns
- [ ] Solution status checked with `getprobstat` before accessing solution values
- [ ] `options noimplicit` considered to enforce explicit declarations
- [ ] Character encoding addressed if source or data files use non-ASCII characters
- [ ] Profiler run planned or documented
- [ ] No debug/trace compile options (`-g`, `-G`) in the production build
- [ ] Constraints/variables named only when necessary; `XPRS_LOADNAMES` left `false` unless needed
- [ ] Temporary/auxiliary data structures reused or cleared rather than recreated
- [ ] Repeated accesses to non-static (`dynamic`/`hashmap`) array elements cached locally via `with` where appropriate
- [ ] Objects declared locally in subroutines unless global scope is required
