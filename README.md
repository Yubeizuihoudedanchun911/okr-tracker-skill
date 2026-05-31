# OKR Tracker Skill

Track OKR progress against weekly reports in Feishu (Lark) docs. Automatically compares your quarterly objectives with reported progress, identifies gaps, and provides prioritized recommendations.

## Features

- **`/okr-setup`**: Create/import quarterly OKR objectives from Feishu OKR system or manual input
- **`/okr-review`**: Weekly progress analysis — reads your Feishu weekly report, compares against OKR targets, outputs gap analysis and actionable suggestions

## Prerequisites

- [lark-cli](https://github.com/anthropics/cli) installed and configured (`lark-cli config init`)
- Claude Code CLI

## Installation

```bash
git clone https://github.com/<your-username>/okr-tracker-skill.git
cd okr-tracker-skill

# Create symlinks
ln -s $(pwd)/okr-setup ~/.claude/skills/okr-setup
ln -s $(pwd)/okr-review ~/.claude/skills/okr-review
ln -s $(pwd)/shared ~/.claude/skills/okr-tracker-shared
```

Or reference from your `~/.claude/CLAUDE.md`:

```markdown
# OKR Tracker
- **okr-setup** (`/path/to/okr-tracker-skill/okr-setup/SKILL.md`) — Create quarterly OKR objectives
- **okr-review** (`/path/to/okr-tracker-skill/okr-review/SKILL.md`) — Weekly OKR progress review
```

## Usage

### 1. Set up OKR objectives (once per quarter)

```
/okr-setup
```

Follow the prompts to enter your quarterly OKRs. Two input methods supported:
- Paste text directly (copy from Feishu OKR system)
- Provide Feishu OKR document URL for automatic import

### 2. Weekly review

```
/okr-review
```

Automatically reads your Feishu weekly report doc, compares against OKR targets, and outputs a progress report with recommendations.

### 3. Scheduled execution (optional)

See [examples/cron-setup.md](examples/cron-setup.md) for scheduling options:
- System crontab
- macOS launchd
- Claude CronCreate (session-scoped)

## Data Storage

All data stored in `~/.claude/okr/`:

| File | Purpose |
|------|---------|
| `config.json` | User config (name, Feishu doc URL, timezone, etc.) |
| `<year>-Q<n>.json` | Quarterly OKR objectives and progress data |

See [examples/sample-okr.json](examples/sample-okr.json) for the data format.

## License

MIT
