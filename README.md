# agent-skills

Agent skills from [Socrait](https://github.com/socrait), packaged as a Claude Code plugin
marketplace. Each plugin is scoped to one stack, and says so in its name.

## Plugins

| Plugin | What it does |
|---|---|
| [`python-backend`](plugins/python-backend/) | Code review for Python web backends built with FastAPI, SQLAlchemy, and Pydantic: `/python-backend:review-all` runs a full review across six specialists — data access, design, security, resilience, testing, and performance — and ends with a merge verdict. |

## Install

In Claude Code, add the marketplace once:

```
/plugin marketplace add socrait/agent-skills
```

Then install the plugins you want:

```
/plugin install python-backend@socrait-agent-skills
```

Each plugin's README covers usage, the conventions its skills assume, and how to adapt them.

## License

[MIT](LICENSE)
