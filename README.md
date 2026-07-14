# orca-council

Convene a council of AI agents on a question via [Orca orchestration](https://www.onorca.dev/docs/cli/orchestration): each member deliberates independently, returns a structured verdict, and a judge — an agent or you — issues the final ruling.

## Prerequisites

- [Orca](https://www.onorca.dev/download) installed and running
- Orchestration enabled: **Settings → Experimental**
- `orca status --json` succeeds from a terminal
- The coordinator agent has Orca's orchestration skill installed:

```bash
npx skills add https://github.com/stablyai/orca --skill orchestration
```

## Install

```bash
npx skills add https://github.com/ashrafabulsoud/orca-council-skill --skill orca-council
```

Installs into your agent's skill directory (Claude Code, Codex, Cursor, etc.).

## Update

Installed skills are copied, not symlinked — they don't pick up repo changes automatically. To get the latest version, re-run the install command:

```bash
npx skills add https://github.com/ashrafabulsoud/orca-council-skill --skill orca-council
```

Or update everything the `skills` CLI manages at once:

```bash
npx skills update
```

## Usage

From Claude Code inside an Orca terminal:

```
/orca-council should we migrate the mobile app to Capacitor?
```

Invoked with no arguments, it asks for the question first. On agents without slash invocation, ask the agent to "use the orca-council skill" and state the question.

### What happens

1. **Preflight** — verifies the Orca runtime and checks for stale council state
2. **Roster** — builds the full agent menu (installed CLIs + Orca-launchable types, with live terminals as context), then asks four questions one at a time: members (2–5) picked from the full menu (never truncated to a widget's option cap), model & effort per seat (skip = each agent's last-used options), judge (`me` or an agent), deliberation depth
3. **Dispatch** — the whole council runs in one dedicated council worktree (a child workspace of where you convened it), one terminal tab per seat; one tracked task per seat, identical spec, enforced verdict trailer:
   ```
   VERDICT: approve | reject | needs-changes | <option>
   CONFIDENCE: low | medium | high
   RATIONALE: <one or two sentences>
   ```
4. **Collection** — waits on `worker_done` messages; blocking questions from members are relayed to you; timeouts are treated as checkpoints, not failures
5. **Judgment** — agent judge gets all verdicts in a synthesis task; `me` gets the verdict table and rules directly
6. **Report** — verdict table + RULING / KEY REASONING / DISSENT WORTH NOTING / NEXT STEP

### Example session

```
> /orca-council GLM-OCR vs dedicated layout model for multi-column Arabic PDFs?

Coordinator: 3 idle terminals found: claude (main), codex (api-work), glm (scratch).
Council members? [suggested: all 3]  → all 3
Judge? [me / agent]                  → me
Repo read access for members?        → yes, docs/ocr-eval/

... dispatching 3 seats ... collecting votes ...

| Member | Agent | Verdict        | Confidence | Rationale                          |
|--------|-------|----------------|------------|------------------------------------|
| 1      | claude| needs-changes  | high       | GLM-OCR fine, layout step missing  |
| 2      | codex | approve        | medium     | throughput beats marginal accuracy |
| 3      | glm   | needs-changes  | high       | column order fails on RTL tables   |

Your ruling?
```

## Notes

- **Cost**: a council = N+1 agent sessions. Keep it for decisions that deserve it.
- `disable-model-invocation: true` means the skill only runs when explicitly invoked (Claude Code). Delete that frontmatter line to allow autonomous convening. Other agents ignore the field.
- The skill never runs `orca orchestration reset` without explicit confirmation — reset is runtime-global and will break other active coordinators.
- Structure follows the Orca skills registry convention (`skills/<name>/SKILL.md`), same as Orca's own `orca-cli` / `orchestration` / `computer-use` skills.

## Layout

```
skills/
└── orca-council/
    └── SKILL.md
```

## License

MIT
