# xpress-ai-helpers

This repo provides guidance documents ("skills") that help LLMs or AI coding agents
work better on tasks related to FICO Xpress Solver, such as writing Xpress Python API models, writing Mosel code, and searching Xpress documentation.

Each skill folder contains plain markdown files, a provider-agnostic format
intended to work with any LLM-based tool. These skills were developed and
tested with [Claude](https://www.anthropic.com/claude-code).

## Skills

| Skill | Purpose |
|---|---|
| [`xpress-python-api`](skills/xpress-python-api/) | Xpress Python API modeling skill |
| [`xpress-mosel`](skills/xpress-mosel/) | Writing, reviewing, and productizing Mosel models |
| [`xpress-docs`](skills/xpress-docs/) | Searching and referencing FICO Xpress documentation |

## Using these skills

Every skill is a folder containing a `SKILL.md` (short YAML frontmatter
block with name + description, followed by the core guidance) and, for
some skills, additional reference files loaded on demand and linked from
`SKILL.md`. The frontmatter is a light convention header as used with Claude, but the files underneath are plain instructions any LLM can read.

There are two ways to use a skill, depending on your tool:

### 1. Tools with native "skill" or "rule file" support

Some agents can load a folder of instruction files automatically and decide
on their own when to apply them (rather than you pasting them in every
time). If your tool supports this pattern, point it at the relevant
`skills/<name>/` folder, or copy that folder into wherever your tool expects
these files to live.

- **Claude Code**: copy the skill folder into `~/.claude/skills/` (user-level)
  or `.claude/skills/` inside your project (project-level), then restart
  Claude Code. It will surface the skill automatically when your prompt
  matches its description, or you can invoke using a slash command ("/[skill-name]").
- **Other agents with folder-based skill/rule conventions**: check your tool's
  docs for the equivalent directory and follow the same pattern -- the
  `SKILL.md` content itself does not need to change.

### 2. Tools without native skill support (most chat-based LLMs)

For any AI assistant you interact with through a chat window:

1. Open the skill's `SKILL.md` in `skills/<name>/`.
2. Copy its content. You can skip or ignore the YAML frontmatter block at
   the top (the `---`-delimited section) -- it is metadata for tools that
   parse it automatically and adds nothing when pasted manually.
3. Paste it into your tool's system prompt, custom instructions, or project
   context, or paste it directly into the chat before asking your question
   if your tool has no persistent instructions feature.

Some skills also include additional reference files alongside `SKILL.md`
(linked from within it) covering more specific sub-topics. `SKILL.md` says
when to pull one in -- copy/paste that file's content too at that point,
the same way you would `SKILL.md` itself.

### Optional: Claude Code plugin marketplace

If you use Claude Code across many projects and want these skills to
auto-update instead of copying folders manually, you can package this repo
as a [Claude Code plugin marketplace](https://docs.claude.com/en/docs/claude-code/plugin-marketplaces)
rather than following the manual copy steps above. This is optional and
Claude-Code-specific -- it is not needed to use the skills, and every skill
still works with the plain copy-paste approach for any other tool.

To set this up: add a `.claude-plugin/marketplace.json` at the repo root
listing each skill folder as a plugin `source`, plus a `.claude-plugin/plugin.json`
inside each skill folder with its name/description/version. Users then run
`/plugin marketplace add <this-repo>` once, and `/plugin install <skill-name>`
per skill, and Claude Code keeps them in sync with this repo going forward.
This repo does not currently ship a marketplace layout.

## Examples

These show the kind of prompt each skill is meant to help with, and what a
good response looks like. Exact invocation depends on your tool (slash
command, named reference, or pasted instructions -- see "Using these
skills" above); the examples below just describe the skill as "loaded"
without assuming a specific syntax.

### xpress-python-api

**Building a model from scratch:**

```text
With the xpress-python-api skill loaded: I have a fleet of 20 vehicles and
50 delivery stops. Build a basic VRP assignment model with binary
variables, demand constraints, and an objective minimizing total distance.
Use indicator constraints to link open/closed routes.
```

Expected: a complete model using `addVariable`, `addConstraint`,
`addIndicator`, `setObjective`, and the correct `SolStatus` enum check
after solving.

**Debugging an infeasible model (pulls in `infeasibility-iis.md`):**

```text
With the xpress-python-api skill loaded: my model keeps returning
INFEASIBLE. Here is the relevant constraint block:
<paste code>
How do I find which constraints are in conflict?
```

Expected: an explanation of `firstIIS(0)` / `getIISData()`, how to decode
constraint names from `getNameList(xp.Namespaces.ROW)`, and a flag on any
deprecated API usage in the pasted code.

### xpress-mosel

**Reviewing a model file:**

```text
With the xpress-mosel skill loaded: review this Mosel file against the
language usage and style guidelines -- MYMODEL.mos
<paste file content>
```

Expected: a check against the naming conventions, structure, and
performance guidance in `SKILL.md` (e.g. named index sets, `exists()`
sparse-loop patterns, constants vs. parameters), with specific line
references for anything that doesn't follow them.

**Writing a new model:**

```text
With the xpress-mosel skill loaded: write a Mosel model for a facility
location problem -- binary open/close variables per facility, continuous
flow variables per facility-customer pair, capacity and demand
constraints, minimize fixed + transport cost.
```

Expected: a model following the section order (constants -> declarations
-> data -> variables -> constraints -> objective), 2-space indentation,
and the naming conventions from Section 3 (e.g. `open`/`flow` for
variables, `CtrCapacity`/`MinCost` for constraints/objective).

### xpress-docs

**Looking up an exact control definition:**

```text
With the xpress-docs skill loaded: what are the valid values and default
for the MIQCPALG control?
```

Expected: a `grep` against `optimizerC.md` for `MIQCPALG`, then the exact
type/range/default quoted from the matched section -- not a recalled
answer.

**Looking up a non-Python API:**

```text
With the xpress-docs skill loaded: what's the R function to load an LP
problem from arrays?
```

Expected: a search of `R-api.md` for the loading function, with its exact
signature quoted from the file.

## Contributing / feedback

This repo is maintained by the FICO Xpress team. Open an issue for feedback
or questions.

## Legal and license requirements

The content in this repository is licensed under the Apache License, Version 2.0. You may not use these files except in compliance with the License. You may obtain a copy of the License at [http://www.apache.org/licenses/LICENSE-2.0](http://www.apache.org/licenses/LICENSE-2.0), or see [LICENSE](LICENSE) for the full license text. Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License.

These files are guidance for AI agents, not a substitute for testing your own work. LLMs can misapply this guidance, generate incorrect code, or make mistakes regardless of how good the underlying instructions are. Always review, test, and validate anything an AI agent generates before relying on it, especially before running it against production systems, licenses, or data. FICO is not liable for outcomes, damages, or losses resulting from the use of these files or of any code, output, or advice an AI agent produces while using them.

These skills describe general practice for using FICO&reg; Xpress with AI agents. They are not official FICO Xpress product documentation. For authoritative reference, see the [FICO&reg; Xpress documentation](https://www.fico.com/fico-xpress-optimization/docs/latest/). Some skills may reference FICO&reg; Xpress software; use of that software is subject to the Community License terms of the [Xpress Shrinkwrap License Agreement](https://www.fico.com/en/shrinkwrap-license-agreement-fico-xpress-optimization-suite-on-premises). See the [licensing options](https://www.fico.com/en/fico-xpress-trial-and-licensing-options) overview for additional details and information about obtaining a paid license.
