# Personal Cursor skills, agents, and rules

Portable Cursor kit: research, code/plan/MR review, refactor planning, CI hygiene, and a generic `/ship` orchestrator. Intended as a **cherry-pick source** for Cursor User Rules (and optional project `.cursor/` copies)—not as a bulk dump into `~/.cursor/`.

## Install (recommended): User Rules

Pick individual items and add them through the Cursor UI so they apply to both local and Cloud agents:

1. Browse this repo and open the specific rule or skill you want (start with [`rules/subagents.mdc`](rules/subagents.mdc)—highest leverage for when to delegate and which models to use).
2. Open **Customize** in the sidebar → **Rules**.
3. **Add Rule** (User Rule) and paste the contents.
4. Repeat for any other rules or skills you want; skip the rest.

Skills from this repo are optional extras when a project already ships its own `.cursor/skills/`.

See [Cursor rules docs](https://cursor.com/docs/rules).

## Alternate: copy into a project

To use as project rules/skills for one repo:

```bash
git clone https://github.com/PhillipChaffee/.cursor.git
cp -R .cursor/skills/* /path/to/your-project/.cursor/skills/
cp -R .cursor/agents/* /path/to/your-project/.cursor/agents/
cp -R .cursor/rules/* /path/to/your-project/.cursor/rules/
```

(Adjust paths if you cloned into a directory not named `.cursor`.) Prefer User Rules for Cloud parity.

## Catalog

### Skills

| Skill | Purpose |
|-------|---------|
| `ship` | Orchestrate ticket → research → plan → implement → verify → MRs |
| `deep-research` | Tiered research with parallel collectors + synthesis |
| `code-review` | Multi-agent code review harness |
| `looping-code-review` | Review → fix → push → re-review loop |
| `plan-review` | Multi-agent plan review |
| `clean-plan` | Slim plans for simple implementer models |
| `mr-review` | GitLab MR review with draft notes |
| `pre-mr-checklist` | Pre-MR verification checklist |
| `ci-lint-test` | Run a project's CI lint/test jobs locally |
| `refactor-planner` | Refactor planning with placement scouts |

### Agents

Code-review (`cr-*`), plan-review (`pr-*`), refactor scouts, and research agents (`researcher-*`, `research-planner`, `research-synthesizer`), including `cr-organization` for file/folder placement.

### Rules

- **Workflow:** `subagents`, `plan-steps`, `engineering`, `merge-requests`, `minimal-changes`, `linear-tickets`
- **Python:** `python`, `python-class-sections`, `python-docstrings-google`, `python-pytest`
- **Tooling:** `github-vs-gitlab-mcp`, `django-migrations`, `comment-style`, `skill-creation`, `code-organization`
- **Docs:** `design-docs` (`alwaysApply: false`), `mr-review-chat-title`
- **Template:** `writing-voice` (`alwaysApply: false`) — fill in before enabling

## Personalizing the writing-voice rule

[`rules/writing-voice.mdc`](rules/writing-voice.mdc) ships as a **template**. Replace every `<...>` placeholder with your own voice, then enable the rule (`alwaysApply: true` or via Settings). Until you fill it in, leave it disabled so it does not inject empty guidance.

## License

MIT — see [LICENSE](LICENSE).
