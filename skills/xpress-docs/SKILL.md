---
name: xpress-docs
description: Search and reference the FICO Xpress Markdown documentation that ships with an Xpress installation, using plain grep/read -- covers the Optimizer C API, control parameters, tuning guide, nonlinear/Global solver, Mosel language, and non-Python language APIs (Java, .NET, R, MATLAB, C++ objects).
when_to_use: Activate when the user asks a detailed reference question about Xpress (an exact function signature, a control's type/range/default, a chapter of the Mosel language reference, etc.) and a quick grep of the installed documentation would answer it more reliably than general knowledge. For Python-specific modeling questions, prefer a dedicated Python API skill first if one is available; use this skill for deeper reference lookups or non-Python languages.
---

# Xpress Documentation Search

Search the Markdown documentation that ships with every Xpress installation. These files are generated from the same source as the PDF/HTML manuals, so they always match your installed version -- no internet access or external index required.

## Finding the Docs Directory

The docs live under the Xpress installation's `docs/` folder. Resolve the install path from the `XPRESSDIR` environment variable if set, otherwise ask the user once. Typical locations:

- **Windows:** `%XPRESSDIR%\docs\` (default install: `C:\xpressmp\docs\`)
- **Linux/macOS:** `$XPRESSDIR/docs/` -- there is no single default install
  location on Linux/macOS the way there is on Windows, so always resolve via
  the `XPRESSDIR` environment variable rather than assuming a path.

If neither the environment variable nor a default path works, ask the user for their Xpress install path rather than guessing.

## Directory Layout

```
docs/
├── MD/                   Getting-started, installation, evaluator guides
├── solver/MD/            Optimizer C API, controls, tuning, nonlinear/Global, other language APIs
└── mosel/MD/             Mosel language reference, user guide, module docs
```

**Note:** not every manual has a Markdown version -- Kalis (constraint programming), BCL, Xpress Workbench, and Xpress Insight documentation currently ship as PDF/HTML only. If a search for one of those topics turns up nothing under `docs/*/MD/`, say so and point the user at the PDF/HTML equivalent instead of guessing a filename.

### Key files under `solver/MD/`

| File | Contents |
|---|---|
| `optimizerC.md` | Full C API reference: every function, plus the complete Control Parameters and Problem Attributes chapters. This is the file to search for a control's exact type/range/default, or a function's signature. |
| `opttuning.md` | Tuning strategy guide: how to invoke the Tuner (Console, C, Mosel, BCL, Workbench), manual tuning overview, multi-threaded solving. |
| `nonlinear.md` | Nonlinear programming and SLP solver reference. |
| `global.md` | Global (spatial branch-and-bound) solver reference. |
| `python-interface.md` | Python API reference (every method, full signature). |
| `R-api.md`, `xpressproblemCxx.md`, `xpressproblemJava.md`, `xpressproblemNET.md`, `matlab.md` | R, C++, Java, .NET, and MATLAB object-oriented API references. |

### Key files under `mosel/MD/`

| File | Contents |
|---|---|
| `moselref.md` | Full Mosel language reference. |
| `moselquickref.md` | Quick syntax reference. |
| `moselug.md` | Mosel user guide. |
| `mosellibs.md` | Mosel C library reference. |
| `moseldata.md` | Working with the database and spreadsheet modules. |
| `moselio.md` | Overview of I/O drivers of the Mosel distribution. |

There are more specialized Mosel files (data-source guides, native-interface guides, per-module references, etc.) -- if `moselref.md`/`moselug.md` don't cover a topic, list the directory (`ls`/`dir` on `mosel/MD/`) to find a more specific file by name.

### Key files under `MD/`

`installation.md` (setup and licensing), `getstart.md` (getting-started tutorial), `evalguide.md`/`evalguideadv.md` (guides for evaluators), `createapp.md` (creating Xpress Solver applications), `mipformref.md` (MIP formulations and linearizations quick reference).

**Do not assume other filenames from the PDF/HTML manual name** (e.g. the Solver C API reference is `optimizerC.md`, not `optimizer.md`) -- list the directory or grep across `*.md` if unsure rather than guessing a filename from the online doc title.

## Search Workflow

These files are large (tens of thousands of lines each), so search before reading, and only read the specific section you find.

### Step 1: Grep for the term

```bash
# Search one file for an exact term (function name, control name, keyword)
grep -n "HEUREMPHASIS" "$XPRESSDIR/docs/solver/MD/optimizerC.md"

# Search across all solver docs at once if you don't know which file has it
grep -rn "XPRSloadlp" "$XPRESSDIR/docs/solver/MD/"

# Case-insensitive, with a few lines of context
grep -in -C 3 "primal ray" "$XPRESSDIR/docs/solver/MD/optimizerC.md"
```

Search tips:
- Function and control entries are formatted as Markdown headings like
  `#### <a id="XPRSloadlp"></a>XPRSloadlp` or `#### <a id="HEUREMPHASIS"></a>HEUREMPHASIS`
  -- searching for the bare name (no `#`) finds both the heading and every
  place the term is mentioned in running text, which is usually what you want.
- If a term isn't found in the file you expected, broaden with `grep -rn` across the whole `solver/MD/` or `mosel/MD/` directory -- topics sometimes live in an unexpected file.
- If you get too many matches, narrow with more specific phrasing or add `-w` for whole-word matching.

### Step 2: Read the surrounding section

Once grep gives you a line number, read a window around it rather than the whole file:

```bash
sed -n '14000,14060p' "$XPRESSDIR/docs/solver/MD/optimizerC.md"
```

Or, if your tool has a file-reading capability with line-range support, use the line number from Step 1 as the offset and read ~40-60 lines forward -- function/control entries are self-contained sections bounded by the next `####`/`###` heading.

### Step 3: Answer with the reference

- Quote the exact signature, control type/range/default, or reference text found -- don't paraphrase from memory once you've found the real entry.
- Cite the file so the user can look further: e.g. "see `optimizerC.md` around the `HEUREMPHASIS` entry."
- If nothing relevant turns up after a reasonable search, say so explicitly rather than answering from general knowledge -- Xpress behavior changes between versions, and a stale general answer is worse than admitting the local docs didn't have it.

## Example Queries

**"What are the valid values for the `MIQCPALG` control?"**
```bash
grep -n "MIQCPALG" "$XPRESSDIR/docs/solver/MD/optimizerC.md"
```
Then read the section at the matched line for the type/range/default table entry.

**"What's the exact C signature of `XPRSaddcuts`?"**
```bash
grep -n "XPRSaddcuts" "$XPRESSDIR/docs/solver/MD/optimizerC.md"
```
Read the `_**Synopsis:**_` block in the matched section.

**"How do I invoke the Tuner from the Mosel language?"**
```bash
grep -n -A 15 "Xpress Mosel" "$XPRESSDIR/docs/solver/MD/opttuning.md"
```

**"What does the Mosel `forall` construct support?"**
```bash
grep -n "forall" "$XPRESSDIR/docs/mosel/MD/moselref.md"
```

**"Does the R interface support IIS?"**
```bash
grep -in "IIS" "$XPRESSDIR/docs/solver/MD/R-api.md"
```

## When to Prefer a Different Skill

- **Python API modeling and quick-reference patterns** (variables, constraints, callbacks, common gotchas): use a dedicated Python API skill first if one is available in your setup. Come back to this skill for the deeper `python-interface.md` reference or when the quick-reference skill doesn't cover something.
- **Mosel language style/conventions, package structure, or code review**: use a dedicated Mosel-modeling skill if available; use this skill to look up specific language reference details it points you to.

This skill exists for the case those don't cover: an exact reference lookup, a non-Python language, or a topic no other skill addresses yet.
