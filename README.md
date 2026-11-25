# agent-supervisor-qc

An [Agent Skills](https://agentskills.io) package that turns any agent into a **Supervisor / QC agent** for multi-agent systems.

It distills a production-proven executor–supervisor quality loop: structured verdicts (approve / rework / escalate), enforced item-by-item rework feedback, risky-action gates, stuck-task watchdogs, and pre-retry state evaluation — with tolerant parsing and fail-closed fallbacks.

## What's inside

```
skills/agent-supervisor-qc/
├── SKILL.md                        # Core loop, verdict protocol, rework enforcement, robustness rules
└── references/
    ├── prompt-templates.md         # 5 copy-ready supervisor prompt templates
    ├── verdict-and-parsing.md      # Verdict vocabulary, scoring rubric, tolerant parsing algorithm
    └── failure-modes.md            # 12 production failure modes and their fixes
```

## Install

**Claude Code** (plugin marketplace):

```
/plugin marketplace add xiaobinger/agent-supervisor-qc
/plugin install agent-supervisor-qc@agent-supervisor-qc
```

**Any skills-compatible agent** (via [skills.sh](https://skills.sh)):

```
npx skills add xiaobinger/agent-supervisor-qc
```

## When it applies

- Reviewing / verifying / accepting another agent's output
- Building executor–supervisor collaboration loops with quality gates
- Second-opinion gating of risky or irreversible tool actions
- Deciding retry vs. abandon for stuck or timed-out AI tasks
- Verifying rework feedback was actually implemented item by item

## License

MIT
