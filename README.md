# Vend16 for Claude Code

Tell your agent "add in-app purchases" and it wires up [Vend16](https://vend16.com): verified App Store and Google Play purchases, subscriptions that stay in sync, refunds you actually hear about. Works with Capacitor, React Native, Flutter, Swift and Kotlin.

## Install (Claude Code)

```
/plugin marketplace add spruikco/vend16-skill
/plugin install vend16@vend16
```

Then, in your app's repo:

```
Add in-app purchases to this app with Vend16
```

## Any other agent (Codex, Cursor, Copilot)

The skill is a plain [Agent Skill](https://agentskills.io) at `plugins/vend16/skills/vend16/SKILL.md`. Copy that folder into your agent's skills directory, or paste this prompt:

```
Integrate Vend16 in-app purchase verification into this app. Read https://vend16.com/llms-full.txt first and follow it exactly.
```

## What you need

A free Vend16 account (first 100 live purchases free, unlimited test purchases): https://vend16.com/signup

The client plugin for Capacitor is open source: https://github.com/spruikco/capacitor-iap

MIT licensed. Made by [Spruik](https://spruik.co).
