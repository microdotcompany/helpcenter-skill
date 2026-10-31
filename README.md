# Helpcenter Skill

An AI agent skill for managing help center articles via the [Helpcenter](https://help.center) API. Works with any AI coding agent that supports the [Agent Skills](https://agentskills.io) specification.

## What it does

Create, update, search, publish, and organize help center articles directly from your AI agent. Just describe what you want — "write a help article about getting started" or "update the FAQ" — and the skill handles the API calls.

**Supported actions:**

- Create and publish articles with well-structured HTML
- Search and update existing articles (preserving unchanged content)
- Manage drafts, categories, translations, and SEO metadata
- Publish/unpublish articles on demand
- Manage knowledge bases and their design, SEO, and custom code
- Read inbox conversations, draft and send replies, and post internal notes
- Manage AI-only FAQ knowledge
- Work through Actions (recurring customer questions the docs don't answer yet)
- Read analytics and query conversation and article metrics

## Installation

### Claude Code

```bash
claude install-skill https://github.com/microdotcompany/helpcenter-skill
```

### OpenClaw

Tell your agent:

```
Install the helpcenter skill from https://github.com/microdotcompany/helpcenter-skill
```

### Codex

Clone into your Codex skills directory:

```bash
git clone https://github.com/microdotcompany/helpcenter-skill.git ~/.agents/skills/helpcenter-skill
```

Or for a project-specific install:

```bash
git clone https://github.com/microdotcompany/helpcenter-skill.git .agents/skills/helpcenter-skill
```

### Manual

Copy `SKILL.md` into your project and point your agent to it.

## Setup

You'll need to create an API key from your Helpcenter dashboard (**Settings > General > API**):

1. **API Key** - Create a new key with the necessary scopes:
   - `content.read` - for searching and reading articles and knowledge
   - `content.write` - for creating and updating articles and knowledge
   - `content.publish` - for publishing articles (optional)
   - `content.delete` - for deleting articles and knowledge (optional)
   - `settings.write` - for managing knowledge bases and adding languages (optional)
   - `conversation.read`, `conversation.write`, `conversation.reply` - for the inbox and Actions (optional)
   - `analytics.read` - for the analytics overview (optional)
2. **Center ID** - Found on the same page

The skill will ask for these when you first use it.

## Usage examples

```
> Write a getting started guide for our app
> Update the billing FAQ with the new pricing
> List all draft articles
> Publish the article about authentication
> Search for articles about onboarding
```

## Skill structure

```
├── SKILL.md          # Skill definition and API reference
├── AGENTS.md         # AI agent guidelines
├── README.md
└── LICENSE
```

Follows the [Agent Skills specification](https://agentskills.io/specification.md).

## License

[MIT](LICENSE)
