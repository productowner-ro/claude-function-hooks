# session.start - one chance to set things up

**Nothing here has been run.** These sketches come from the type declarations of a feature that shipped hours ago. Try one, break it, tell everyone what happened.

---

This event fires once, when the session is ready, and the engine waits for it before your first message goes anywhere. **You** are the person at the keyboard. **The hook** is the code you install.

| | |
|---|---|
| The hook receives | `{ cwd, surface, interactive }` |
| Fires | once, before the first prompt |

This is also the only correct place to start work that must outlive a single message. A timer started here with `$.clock.every` keeps running. A timer started inside a normal hook gets cut off when that message is done.

### "Why not just write it in CLAUDE.md?"

`CLAUDE.md` cannot run a command, check a clock, or start a background job. This can.

---

## 1. Warn when your local copy is stale

You ask a question about code in another repository. Your copy of it is eleven days old. You get a confident answer about code that no longer exists.

The hook checks each repository at startup and tells you which ones are behind.

```ts
on('session.start', async ($, e, next) => {
  for (const repo of ['../api', '../web']) {
    await $.process.run(['git', '-C', repo, 'fetch', '--quiet'])
    const behind = await $.process.run(['git', '-C', repo, 'rev-list', '--count', 'HEAD..@{u}'])
    const n = Number(behind.stdout.trim())
    if (n > 0) $.ui.toast(`${repo} is ${n} commits behind. Run git pull.`)
  }
  return next(e)
})
```

**Why not check yourself?** You will not, and the answer looks correct either way.

**The bill.** One fetch per repository, before your first message. Keep the list short or the session feels slow to start.

**The trap.** The engine waits for this hook. Slow work here is a slow startup, every time.

---

## 2. Keep your next meeting on the screen

You start a session at 10:40. Your stand-up is at 11:00. At 11:25 you notice.

The hook puts the next meeting in the status line and updates it every ten minutes.

```ts
on('session.start', async ($, e, next) => {
  const show = async () => {
    const res = await $.http.fetch('http://localhost:7777/next-meeting')
    $.ui.status(res.ok ? `Next: ${res.text}` : undefined)
  }
  await show()
  $.clock.every(10 * 60 * 1000, show)
  return next(e)
})
```

The status line is not part of the conversation. It costs no tokens and interrupts nothing.

**Why not a calendar alert?** Because you are in a terminal, and the alert is on a different screen behind three windows.

**The bill.** One small request every ten minutes.

**The trap.** Timers die when the module reloads. During development that happens on every save.

---

## 3. Register your own tools before turn one

You built a tool for Claude to call. If you register it late, the first message does not know it exists.

The hook registers it here, and the engine waits.

```ts
on('session.start', async ($, e, next) => {
  await $.tool.register({
    name: 'today',
    description: "Return today's task list as text.",
    inputSchema: { type: 'object', properties: {} },
  })
  return next(e)
})
```

The tool is listed as `mcp__<your plugin>__today` from the first message on.

**The bill.** Nothing.

**The trap.** Registering the tool is half the job. You also need a `tool.call` hook that matches that name and returns a result. Without it, the call fails.

---

## 4. Start a session with the work already done

You open a session every morning and ask the same three questions before the real work starts.

The hook does the fetching first and hands you the answer through the status line and a stored value.

```ts
on('session.start', async ($, e, next) => {
  const repo = await $.session.repo()
  if (repo === null) return next(e)
  const failing = await $.http.fetch(`https://ci.example.com/api/failing?repo=${repo.name}`)
  await $.store.set('ci', failing.text)
  if (failing.text !== '0') $.ui.toast(`${failing.text} builds are failing.`)
  return next(e)
})
```

Anything you store here is readable by your other hooks for the rest of the session.

**Why not ask?** Because asking costs a turn, and the answer is the same every morning.

**The bill.** One request at startup.

**The trap.** A service that is down makes your session hang. Set a timeout on `$.http.fetch` work and carry on without it.

---

**Now go argue with this page.** It was written by someone who had the feature for an afternoon. A sketch that turns out wrong is worth more than the page is.
