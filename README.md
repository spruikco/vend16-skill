# Vend16 for Claude Code

Put a buy button in your app. We get the rest sorted.

Tell your agent "add in-app purchases" and it wires up [Vend16](https://vend16.com): verified App Store and Google Play purchases, subscriptions that stay in sync, refunds you actually hear about. Works with Capacitor, React Native, Expo, Flutter, Swift and Kotlin.

## Install (Claude Code)

```
/plugin marketplace add spruikco/vend16-skill
/plugin install vend16@vend16
```

Then, in your app's repo:

```
Add in-app purchases to this app with Vend16
```

The plugin brings two things:

- **The Vend16 skill**: the whole workflow, the money rules, and copy-ready code for each platform in `references/`.
- **The Vend16 MCP server** (`https://vend16.com/mcp`): Claude signs you in through your browser, creates your app, fetches its key and checks your store setup itself, so you never copy a key by hand.

## Any other agent (Codex, Cursor, Copilot)

The skill is a plain [Agent Skill](https://agentskills.io) at `plugins/vend16/skills/vend16/`. Copy that folder into your agent's skills directory, or paste this into your agent:

```
Add in-app purchases to this app with Vend16. Read https://vend16.com/llms-full.txt first and follow it exactly. Run `npx github:spruikco/vend16-cli init --yes --json` to sign me in and set up the app, show me the sign-in code it prints, then wire purchases in following the guide it links.
```

The MCP server works in any agent that supports remote MCP servers with OAuth: add `https://vend16.com/mcp`.

## The pieces

| | |
| --- | --- |
| [vend16-cli](https://github.com/spruikco/vend16-cli) | `npx github:spruikco/vend16-cli init`: sign in, create the app, add the key, install the client, list what is left |
| [capacitor-iap](https://github.com/spruikco/capacitor-iap) | Open-source StoreKit 2 and Play Billing plugin for Capacitor 8 |
| [llms-full.txt](https://vend16.com/llms-full.txt) | The full integration guide, written for agents |
| [vend16.com/docs](https://vend16.com/docs) | The same, for people |

## What you need

A free Vend16 account (first 100 live purchases free, unlimited test purchases). The agent can make one with you, or sign up at https://vend16.com/signup.

MIT licensed. Made by [Spruik](https://spruik.co). Questions: hello@spruik.co.
