# 语言约定

- 沟通：与用户的一切交流使用中文——回复、提问、状态汇报、总结、风险提示。
- 思考：内部推理、任务拆解、方案权衡、风险分析使用中文。

# 代码风格

- 不要写注释

## Agent skills

### Issue tracker

Issues live in this repo's GitHub Issues, managed via the `gh` CLI. See `docs/agents/issue-tracker.md`.

### Triage labels

Default five-role vocabulary: `needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`. See `docs/agents/triage-labels.md`.

### Domain docs

Single-context: root `CONTEXT.md` + `docs/adr/`. See `docs/agents/domain.md`.
