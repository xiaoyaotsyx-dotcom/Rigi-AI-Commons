# Contributing to Rigi AI Commons

Thanks for your interest in contributing! 🎉

## How to Contribute

### 1. Report a Bug / Request a Feature

Open an [Issue](https://github.com/xiaoyaotsyx-dotcom/Rigi-AI-Commons/issues). Use the template.

### 2. Submit a New Skill

We love new workflows! To submit a skill:

1. Fork the repo
2. Create a folder under the skill category: `your-skill-name/`
3. Include:
   - `SKILL.md` — the main skill file (follow existing format)
   - `README.md` — what it does, how to use it
   - Any required scripts in `scripts/`
4. Open a PR

### 3. Improve Existing Skills

- Bug fixes → open PR directly
- Feature changes → open an Issue first to discuss

### 4. Improve Documentation

Docs PRs are always welcome. Typos, clarifications, translations — just PR.

---

## Skill File Format

```markdown
---
name: your-skill-name
description: One-line description of what this does
domain: ecommerce | investing | legal | etc.
triggers:
  - keywords that trigger this skill
---

# Skill Title

## ⚠️ Rules
1. ...
2. ...

## Workflow Steps
### Step 1: ...
### Step 2: ...

## Common Failures
| Symptom | Fix |
|---------|-----|
```

---

## Code Style

- **Python:** Follow PEP 8. Use type hints where helpful.
- **Markdown:** Keep it readable. Bilingual (EN/ZH) preferred for user-facing content.
- **Commit messages:** `type: short description` (e.g., `fix: CDP timeout`, `feat: add ERP form filler`)

---

## PR Checklist

- [ ] Skill loads without errors
- [ ] Tested on Hermes Agent
- [ ] README updated
- [ ] No hardcoded credentials or API keys
- [ ] Bilingual if user-facing

---

## Questions?

Open a Discussion or reach out on RedNote: @瑞吉AI人民公社
