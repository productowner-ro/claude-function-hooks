# tool.call - standing between Claude and the real world

**Nothing here has been run.** These sketches come from the type declarations of a feature that shipped hours ago. Try one, break it, tell everyone what happened.

---

Every command, every file write, every API call goes through one event. **You** are the person at the keyboard. **The hook** is the code you install. It sees the tool call before it happens, and it can answer instead of it, change it, or refuse it.

| | |
|---|---|
| The hook receives | `{ tool, tool_use_id, ...the tool's arguments }` |
| The hook hands back | the result, or `{ deny: "why" }` |

### "Why not just write it in CLAUDE.md?"

"Never run destructive commands on production" is a sentence. Claude reads it and usually follows it. A hook is code. It runs every time, and it cannot forget.

---

## 1. Stop a kubectl command from hitting production

You ask Claude to restart a service. The command is correct. Your terminal is pointed at the production cluster instead of staging, because you switched an hour ago and forgot.

The hook reads the active cluster before the command runs. If the name contains "prod", it asks you first.

```ts
const RISKY = /\b(delete|drain|scale|rollout restart)\b/

on('tool.call', { tool: 'Bash' }, async ($, e, next) => {
  if (!e.command.startsWith('kubectl') || !RISKY.test(e.command)) return next(e)
  const { stdout } = await $.process.run(['kubectl', 'config', 'current-context'])
  if (!stdout.includes('prod')) return next(e)
  const answer = await $.ui.ask(`This runs on ${stdout.trim()}. Continue?`, ['No', 'Yes'])
  return answer === 'Yes' ? next(e) : { deny: 'You stopped it. Wrong cluster.' }
})
```

The hook does not ask Claude which cluster it is on. It asks the machine.

This is the last line of defence. Two other pages cover the same risk earlier: [ui-render.md](ui-render.md) keeps the cluster name on your screen, and [ui-select.md](ui-select.md) lets you switch cluster with a confirmation. Together they mean you see it, you change it deliberately, and the command still checks.

**Why not a rule in `CLAUDE.md`?** A rule tells Claude to check. The hook checks.

**The bill.** One subprocess, only on commands that match.

**The trap.** A hook that crashes is skipped and the turn carries on without it. A guard must be tested throwing.

---

## 2. Get deleted files back

You ask Claude to clean up a folder. It runs `rm` on one file more than you meant. There is no undo.

The hook turns every `rm` into a move. The files go to a dated folder under `.trash/`.

```ts
on('tool.call', { tool: 'Bash' }, async ($, e, next) => {
  if (!/^rm\s/.test(e.command)) return next(e)
  const bin = `.trash/${Date.now()}`
  await $.process.run(['mkdir', '-p', bin])
  const paths = e.command.replace(/^rm\s+(-\w+\s+)*/, '')
  await $.process.run(['sh', '-c', `mv ${paths} ${bin}/`])
  return { result: `Moved to ${bin} instead of deleting. Restore with mv.` }
})
```

Claude reads "the files are deleted" and moves on. The files are still there.

**Why not a rule?** You need the rule to hold on the one day it does not.

**The bill.** Disk space. Empty the folder weekly.

**The trap.** The result text is what Claude reads next. Say plainly what happened, or it will assume the file is gone and plan around a ghost.

---

## 3. Shrink huge tool output before Claude reads it

You ask a question. The tool answers with 40,000 characters of JSON to say six things. Claude reads all of it, and you pay for all of it.

The hook catches the result on its way back and shortens it first.

```ts
on('tool.call', async ($, e, next) => {
  const out = await next(e)
  const text = JSON.stringify(out.result ?? '')
  if (text.length < 8000) return out
  const small = await $.model.complete({
    model: 'claude-haiku-4-5-20251001',
    system: 'Compress to the facts. Keep every id, number and name. Drop nothing that identifies a row.',
    prompt: text,
  })
  return { ...out, result: small }
})
```

Same facts, a fraction of the words. You can drop the model call and convert the JSON to a compact table format instead. That version is free and always gives the same output.

**The bill.** One small model call per oversized result. The turn waits for it.

**The trap.** Compression that drops an id is data loss. List what must survive. Never compact a result Claude has to quote back exactly.

---

## 4. Fix the same mistake every time it happens

Your project has one parameter that must always be passed. Claude passes it most of the time. When it forgets, you get an answer from the wrong database and you do not always notice.

The hook adds the parameter when it is missing.

```ts
on('tool.call', { tool: 'mcp__metabase__execute_query' }, ($, e, next) => {
  if (e.instance !== undefined) return next(e)
  $.ui.log('added the missing instance')
  return next({ ...e, instance: 'production' })
})
```

You never say "run that again, with the right database". The call goes out correct.

**Why not a rule in `CLAUDE.md`?** The rule costs tokens in every session and works in most of them. This costs nothing and works in all of them.

**The bill.** Nothing.

**The trap.** A silent fix hides a real problem. Log it, so you can see how often the model needed rescuing.

---

## 5. Answer "what did Claude actually do?"

Your security team asks what the tool touched last Tuesday. Today you answer that by scrolling through a transcript.

The hook writes one line per tool call to a file.

```ts
on('tool.call', async ($, e, next) => {
  const started = $.clock.now()
  const out = await next(e)
  const line = { at: started, tool: e.tool, input: e, ms: $.clock.now() - started,
                 denied: out.deny !== undefined, session: await $.session.id() }
  await $.fs.writeFile(`.audit/${new Date().toISOString().slice(0, 10)}.jsonl`, JSON.stringify(line) + '\n')
  return out
})
```

Every call, its arguments, how long it took, and whether something blocked it. You can hand the file to someone.

This records what Claude asked to run. Three other pages record different things: [engine-create.md](engine-create.md) records what every plugin read from disk, [turn-complete.md](turn-complete.md) records what changed in folders you care about, and [agent-spawn.md](agent-spawn.md) records what each subagent was told to do. Pick the one that answers the question you actually get asked.

**Why not the transcript?** The transcript is a conversation. This is a list you can search, count and filter.

**The bill.** One file append per call.

**The trap.** Arguments contain secrets. Remove them before writing, or the audit file becomes the leak.

---

**Now go argue with this page.** It was written by someone who had the feature for an afternoon. A sketch that turns out wrong is worth more than the page is.
