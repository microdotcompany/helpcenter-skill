# Helpcenter Skill

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill for managing help center articles via the [Help.Center](https://help.center) API.

## What it does

This skill lets you create, update, search, publish, and organize help center articles directly from Claude Code. Just describe what you want — "write a help article about getting started" or "update the FAQ" — and the skill handles the API calls.

**Supported actions:**

- Create and publish articles with well-structured HTML
- Search and update existing articles (preserving unchanged content)
- Manage drafts, categories, and SEO metadata
- Publish/unpublish articles on demand

## Installation

```bash
claude install-skill https://github.com/microdotcompany/helpcenter-skill
```

## Setup

You'll need two things from your Help.Center dashboard (**Settings > General > API**):

1. **API Key**
2. **Center ID**

The skill will ask for these when you first use it.

## Usage examples

```
> Write a getting started guide for our app
> Update the billing FAQ with the new pricing
> List all draft articles
> Publish the article about authentication
```

## License

[MIT](LICENSE)
