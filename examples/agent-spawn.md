# agent.spawn - the brief a subagent gets

**Nothing here has been run.** These sketches come from the type declarations of a feature that shipped hours ago. Try one, break it, tell everyone what happened.

---

When Claude starts a subagent, it writes the brief itself. That brief is different every time, because Claude writes it fresh. This event hands it to you first. **You** are the person at the keyboard. **The hook** is the code you install.

| | |
|---|---|
| The hook receives | `{ tool_use_id, prompt, description, subagentType, model?, parentModel, permissionMode?, background, fork, name?, cwd? }` |

`prompt` is the brief. You can add to it, replace it, or refuse the spawn.

### "Why not just write it in CLAUDE.md?"

The subagent does not read your `CLAUDE.md` instructions about how to brief subagents. The parent does, sometimes.

---

## 1. Make every brief carry the same standing rules

You want every subagent to report what it found rather than a pass or fail verdict. Claude includes that instruction about half the time.

The hook appends it to every brief.

```ts
const RULES = `
Report what you found and your judgement of it. Do not report a score or a pass/fail.
Name the specific harms you looked for. Quote the evidence.
If you found nothing, say so plainly.`

on('agent.spawn', ($, e, next) =>
  next({ ...e, prompt: `${e.prompt}\n\n${RULES}` }),
)
```

Every agent, every time, identical wording.

**Why not put it in the agent definition?** You can, for agents you wrote. This also covers the built-in ones and anything installed later.

**The bill.** A few words per spawn.

**The trap.** Every word you add competes with the actual task. Keep it to the rules that change the output.

---

## 2. Stop the fourth agent

Claude decides the job splits into six parts and starts six agents. Your machine slows down and the bill goes up.

The hook counts the running ones and refuses past a limit.

```ts
on('agent.spawn', async ($, e, next) => {
  const running = ((await $.store.get('running')) as number) ?? 0
  if (running >= 3) return { deny: 'Three agents already running. Wait for one to finish.' }
  await $.store.set('running', running + 1)
  const out = await next(e)
  await $.store.set('running', Math.max(0, (((await $.store.get('running')) as number) ?? 1) - 1))
  return out
})
```

**Why not a rule?** "Do not start more than three agents" is read once and forgotten by the time the sixth looks reasonable.

**The bill.** Nothing.

**The trap.** If a spawn fails in a way that skips your decrement, the counter sticks and blocks everything. Reset it at `session.start`.

---

## 3. Keep a copy of every brief

An agent comes back with something strange. You want to see what it was actually asked, and that brief is buried.

The hook writes each one to a file.

```ts
on('agent.spawn', async ($, e, next) => {
  const file = `logs/agents/${e.tool_use_id}.md`
  await $.fs.writeFile(file, `# ${e.subagentType}: ${e.description}\n\n${e.prompt}\n`)
  return next(e)
})
```

Now you can read what Claude asked for, in its own words, and fix the instruction that produced it.

**Why not the transcript?** Subagent briefs are long and collapse in the display. One file each is easier to read and to compare.

**The bill.** One file write per spawn.

**The trap.** Briefs quote your work. Keep the folder out of git.

---

## 4. Choose a cheaper model for small jobs

Every subagent gets a capable model, including the one whose job is to list filenames.

The hook downgrades the obvious cases.

```ts
const SIMPLE = /^(list|find|count|check whether)/i

on('agent.spawn', ($, e, next) => {
  if (!SIMPLE.test(e.description)) return next(e)
  return next({ ...e, model: 'claude-haiku-4-5-20251001' })
})
```

**Why not set it per agent type?** Because the same agent type does both easy and hard jobs.

**The bill.** Negative.

**The trap.** A job that looks simple in one sentence can be hard. Log the downgrades and read a few.

---

**Now go argue with this page.** It was written by someone who had the feature for an afternoon. A sketch that turns out wrong is worth more than the page is.
