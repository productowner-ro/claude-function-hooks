# engine.create - adding your own capability to the engine

**Nothing here has been run.** These sketches come from the type declarations of a feature that shipped hours ago. Try one, break it, tell everyone what happened.

---

Every hook receives `$`, the engine interface. `$.fs` reads files. `$.http` makes requests. `$.model` calls a model. This event is the one that builds `$` in the first place. **You** are the person at the keyboard. **The hook** is the code you install.

| | |
|---|---|
| The hook hands back | additions to `$`, as `{ noun: { method } }` |

Two things you can do here: add a noun that did not exist, or wrap one that did.

Here is the part that makes it interesting. Every method on `$` is itself an event. So when you add `$.jira.issue`, you have not just written a function. You have created an event that any other hook can sit on top of: to cache it, log it, rewrite its arguments, or refuse it. Your capability gets the same treatment as the built-in ones, for free.

### "Why not just write it in CLAUDE.md?"

This is not about instructing Claude. It is about what the plugins on your machine are able to do.

---

## 1. Add a capability your other hooks can use

Four of your hooks call Jira. Each one repeats the same MCP call, the same error handling, the same field names. When Jira changes, you edit four hooks.

The hook adds a noun.

```ts
on('engine.create', ($, e, next) => ({
  jira: {
    async issue(key: string) {
      const res = await $.mcp.call('atlassian', 'getJiraIssue', { key })
      return { key, summary: res.summary, status: res.status }
    },
  },
}))
```

Now any hook writes `await $.jira.issue('ABC-1')`. One place knows how Jira works.

**Why not a shared function?** A function is not an event. This is.

**The bill.** Nothing.

**The trap.** You are adding to a shared surface. Pick a name nobody else will pick.

---

## 2. Put a cache in front of your own capability

The same ticket is fetched six times in one session.

Because `$.jira.issue` is an event, a second hook can wrap it. No change to the first one.

```ts
on('jira.issue', async ($, e, next) => {
  const hit = await $.store.get(`jira:${e.key}`)
  if (hit !== undefined) return hit
  const fresh = await next(e)
  await $.store.set(`jira:${e.key}`, fresh)
  return fresh
})
```

This is the same shape as every other hook on this site: before, after, instead, modifying. Your own capability behaves like a built-in one.

**The bill.** Nothing.

**The trap.** A cache with no expiry serves stale answers all day. Store the time and check it.

---

## 3. Record every file the plugins read and write

Your security team asks which files the tool touched. `$.fs` is used by every plugin on the machine, including ones you did not write.

The hook wraps it.

```ts
on('engine.create', ($, e, next) => {
  const base = next(e)
  return {
    ...base,
    fs: {
      ...base.fs,
      async readFile(path: string) {
        await base.fs.writeFile('logs/fs-audit.log', `read ${path}\n`)
        return base.fs.readFile(path)
      },
    },
  }
})
```

Every plugin below yours now goes through your version, whether or not it knows.

**Why not trust the plugins?** Because you did not write them, and the question was not about trust.

**The bill.** One append per file read.

**The trap.** Your wrapper calls `$.fs` itself. Use the original, as the sketch does, or you build an infinite loop.

---

## 4. Give your team's tools a single front door

Your repository has six scripts. Everyone remembers a different subset. Each hook that uses one repeats the path and the arguments.

The hook wraps them all in one noun.

```ts
on('engine.create', ($, e, next) => ({
  repo: {
    async openWork() {
      const out = await $.process.run(['python3', 'tools/board.py', '--open'])
      return JSON.parse(out.stdout)
    },
    async validate(file: string) {
      const out = await $.process.run(['python3', 'tools/validate.py', file])
      return { ok: out.exitCode === 0, problems: out.stderr }
    },
  },
}))
```

`$.repo.openWork()` reads better than a subprocess call, and when the script moves you change one line.

**Why not call the scripts directly?** Because six hooks calling six scripts means six places to update.

**The bill.** Nothing.

**The trap.** A method that hides a slow script is still slow. Say so in the name or the comment.

---

**Now go argue with this page.** It was written by someone who had the feature for an afternoon. A sketch that turns out wrong is worth more than the page is.
