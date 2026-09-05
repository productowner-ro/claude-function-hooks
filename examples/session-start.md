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

## 3. Give Claude different tools in different folders

You have twelve tools. In your documentation folder, nine of them are useless. Claude still reads a description of all twelve in every session, and sometimes picks a wrong one.

The hook registers only the tools that fit the folder you opened.

```ts
const BY_FOLDER: Record<string, ToolSpec[]> = {
  '/docs': [
    { name: 'wordcount', description: 'Count words in a file.', inputSchema: { type: 'object', properties: { path: { type: 'string' } } } },
    { name: 'readability', description: 'Score a draft for reading level.', inputSchema: { type: 'object', properties: { path: { type: 'string' } } } },
  ],
  '/infra': [
    { name: 'costEstimate', description: 'Estimate the monthly cost of a change.', inputSchema: { type: 'object', properties: {} } },
  ],
}

on('session.start', async ($, e, next) => {
  const tools = Object.entries(BY_FOLDER).find(([dir]) => e.cwd.includes(dir))?.[1] ?? []
  for (const tool of tools) await $.tool.register(tool)
  return next(e)
})
```

The engine waits for this event, so every tool you register here is listed from the first message. Registering the same name again replaces it.

You can also decide from the repository rather than the path. `$.fs.ancestors({ names: ['package.json', 'Makefile'] })` walks up the tree, so a folder with a `Makefile` gets your build tools and one without does not.

**Why not install every tool everywhere?** Each tool's description is read in every session. Twelve tools is a paragraph you pay for constantly and use three of.

**The bill.** Negative. You removed nine descriptions from every session in that folder.

**The trap.** Registering the tool is half the job. You also need a `tool.call` hook matching `mcp__<your plugin>__<name>` that returns a result. Without it, the call fails and says so.

To register a tool part-way through a session, based on what you just asked for, see [prompt-submit.md](prompt-submit.md).

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

Anything you store here is readable by your other hooks for the rest of the session. The dashboard in [ui-render.md](ui-render.md) reads this `ci` key.

Use this event when the fact should reach you. Use [prompt-context.md](prompt-context.md) when the fact should reach Claude.

**Why not ask?** Because asking costs a turn, and the answer is the same every morning.

**The bill.** One request at startup.

**The trap.** A service that is down makes your session hang. Set a timeout on `$.http.fetch` work and carry on without it.

---

**Now go argue with this page.** It was written by someone who had the feature for an afternoon. A sketch that turns out wrong is worth more than the page is.
