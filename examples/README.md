# One page per event, with things you could build on it

**Nothing in this folder has been run.** Every sketch was written from the type declarations of a feature that shipped hours ago. They are ideas with code attached, not working plugins. Try one. When it breaks, or works, say so.

A hook is a small piece of code that sits between you and Claude. It runs before Claude reads your message, or before a command runs, or before something is drawn on your screen. It can change what happens, or stop it.

The reason to use one instead of writing a rule in `CLAUDE.md`: a rule is a request that costs tokens in every session, and Claude decides whether to follow it. A hook is code. It runs every time.

## Your message

| Event | What it touches | Examples |
|---|---|---|
| [prompt.submit](prompt-submit.md) | your message before Claude reads it | look up a name, fix a date, hide a password, attach a file |
| [prompt.section](prompt-section.md) | Claude's standing instructions | tone per folder, a late reminder, remove a section, test a rule |
| [prompt.context](prompt-context.md) | the pages attached to your first message | branch state, open work, date format, remove your email |

## Tools and commands

| Event | What it touches | Examples |
|---|---|---|
| [tool.call](tool-call.md) | every command and file write | stop a production command, a recycle bin, shrink huge output, an audit file |
| [tool.describe](tool-describe.md) | what Claude reads before picking a tool | put the rule on the tool, show its cost, retire it, add house rules |
| [PreToolUse](pre-tool-use.md) | the older permission check | ask only when it matters, block late deploys, classify a command |

## The session and the turn

| Event | What it touches | Examples |
|---|---|---|
| [session.start](session-start.md) | once, before your first message | stale repository warning, next meeting, register a tool |
| [turn.start](turn-start.md) | each turn beginning | length warning, a work log, notice repetition |
| [turn.step](turn-step.md) | each reply Claude produces | shorten long answers, count rule breaks, catch a cut-off answer |
| [turn.complete](turn-complete.md) | the finished turn | check written files, a retro log, a sound when it ends |

## Helpers and skills

| Event | What it touches | Examples |
|---|---|---|
| [agent.offer](agent-offer.md) | which subagents Claude sees | remove the broad search, filter by folder, demo mode |
| [agent.spawn](agent-spawn.md) | the brief a subagent gets | one standing brief, cap the count, keep every brief |
| [skill.prompt](skill-prompt.md) | a skill's instructions | change one line without forking, add repository rules, test a change |
| [attribution.text](attribution-text.md) | commits and pull requests | ticket key on every commit, remove the footer, suggest reviewers |

## The screen

| Event | What it touches | Examples |
|---|---|---|
| [ui.render](ui-render.md) | everything drawn | short answers on screen only, production banner, fold output, a dashboard |
| [ui.resolve](ui-resolve.md) | the elements other plugins draw with | one palette, disable buttons, force a width |
| [ui.press](ui-press.md) | a button | convert a file, start a review, approve without a message |
| [ui.input](ui-input.md) | a text field | search as you type, warn while writing, save a draft |
| [ui.select](ui-select.md) | a dropdown | switch cluster safely, pick a reviewer, change mode |

## The engine itself

| Event | What it touches | Examples |
|---|---|---|
| [engine.create](engine-create.md) | the `$` interface every hook receives | add your own capability, cache it, audit every file read |

---

## Before you try one

1. Run `/plugin-types` in a session. It writes the real type declarations from your build. Those are the truth; this folder is a sketchbook.
2. Run `claude --plugin-dir <folder>` to load a plugin from disk for one session.
3. Run with `claude --debug`. A hook that fails is skipped silently, and the debug log is the only place that says why.

The reference for how all of this fits together is [../README.md](../README.md).
