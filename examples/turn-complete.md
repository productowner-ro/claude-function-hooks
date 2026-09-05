# turn.complete - the turn is over and everything is known

**Nothing here has been run.** These sketches come from the type declarations of a feature that shipped hours ago. Try one, break it, tell everyone what happened.

---

This event fires when a turn finishes. Every file it wrote, every command it ran, every answer it gave is already decided. **You** are the person at the keyboard. **The hook** is the code you install.

| | |
|---|---|
| The hook receives | the completed turn, including whether Claude refused |

This is the cleanup and bookkeeping event. It is too late to change anything, which makes it a safe place to work.

### "Why not just write it in CLAUDE.md?"

You would be asking Claude to remember to do something at the end of every turn. It will, most of the time.

---

## 1. Check the files that were just written

Your repository has rules: ticket keys must be links, dates must use one format, cited paths must exist. Today somebody runs the checker later, or nobody does.

The hook runs it on every file the turn touched.

```ts
on('turn.complete', async ($, e, next) => {
  const out = await next(e)
  const changed = await $.process.run(['git', 'diff', '--name-only'])
  for (const file of changed.stdout.split('\n').filter((f) => f.endsWith('.md'))) {
    const check = await $.process.run(['python3', 'tools/validate.py', file])
    if (check.exitCode !== 0) $.ui.toast(`${file}: ${check.stderr.slice(0, 120)}`)
  }
  return out
})
```

**Why not run it yourself?** You would have to remember, on every turn, forever.

**The bill.** One subprocess per turn that changed a markdown file.

**The trap.** A checker that is slow makes every turn feel slow. Only run it on files that changed.

---

## 2. Keep a log for the retrospective

Two weeks later somebody asks what changed in a sensitive folder and why. You scroll.

The hook writes one line per turn, but only for folders you care about.

```ts
const WATCHED = ['migrations/', 'config/', 'infra/']

on('turn.complete', async ($, e, next) => {
  const out = await next(e)
  const changed = (await $.process.run(['git', 'diff', '--name-only'])).stdout.split('\n')
  const hits = changed.filter((f) => WATCHED.some((w) => f.startsWith(w)))
  if (hits.length === 0) return out
  const line = `${new Date($.clock.now()).toISOString()} | ${hits.join(', ')} | ${e.text?.slice(0, 100)}\n`
  await $.fs.writeFile('logs/watched-changes.log', line)
  return out
})
```

The file answers three questions: what changed, when, and what was asked for. That is most of a retrospective.

This is one of four logging examples in this folder, and they answer different questions. [tool-call.md](tool-call.md) records every command. [engine-create.md](engine-create.md) records every file read. [agent-spawn.md](agent-spawn.md) records every subagent brief. This one records what changed and why.

**Why not read the git log?** Commit messages describe the code. This records the intent behind it, in your words.

**The bill.** One subprocess and one append per turn.

**The trap.** `$.fs.writeFile` replaces the file. Read it first and append, or you keep one line.

---

## 3. Tell you the turn is finished

You start a long turn and switch to your email. It finishes. You come back nine minutes later.

The hook makes a sound when a slow turn ends.

```ts
on('turn.complete', async ($, e, next) => {
  const out = await next(e)
  const started = (await $.store.get(`started:${e.turnId}`)) as number | undefined
  if (started !== undefined && $.clock.now() - started > 120000) {
    await $.audio.speak('Finished.')
    await $.store.delete(`started:${e.turnId}`)
  }
  return out
})
```

Pair it with the `turn.start` sketch that records the start time.

**Why not watch the screen?** Because waiting is the reason you left.

**The bill.** Nothing until it fires.

**The trap.** In an open office this is a bad idea. Use `$.ui.toast` there.

---

## 4. Record when Claude refuses

Claude declines a request. You rephrase and move on. Nobody counts how often this happens or on what.

The hook keeps the record.

```ts
on('turn.complete', async ($, e, next) => {
  const out = await next(e)
  if (e.refused) {
    const line = `${new Date($.clock.now()).toISOString()} | ${e.text?.slice(0, 200)}\n`
    await $.fs.writeFile('logs/refusals.log', line)
  }
  return out
})
```

If the same kind of request keeps failing, that is a fact about your setup, not a mood.

**The bill.** Nothing.

**The trap.** This file records what you asked for. Treat it like any other log that quotes your input, and keep it out of git.

---

**Now go argue with this page.** It was written by someone who had the feature for an afternoon. A sketch that turns out wrong is worth more than the page is.
