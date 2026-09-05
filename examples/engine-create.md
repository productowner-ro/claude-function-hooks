# engine.create - adding your own capability to the engine

**Nothing here has been run.** These sketches come from the type declarations of a feature that shipped hours ago. Try one, break it, tell everyone what happened.

---

Every hook receives `$`, the engine interface. `$.fs` reads files. `$.http` makes requests. `$.model` calls a model. This event is the one that builds `$` in the first place. **You** are the person at the keyboard. **The hook** is the code you install.

| | |
|---|---|
| The hook hands back | additions to `$`, as `{ noun: { method } }` |

Two things you can do here. Add a name that did not exist, such as `$.jira`. Or wrap one that did, such as `$.fs`.

The part worth understanding: every method on `$` is an event. When you add `$.jira.issue`, you did not write a function. You created an event called `jira.issue`, and any hook can now sit on top of it and cache it, log it, change its arguments or refuse it. Example 2 does exactly that.

### "Why not just write it in CLAUDE.md?"

This is not about instructing Claude. It is about what the plugins on your machine are able to do.

---

## 1. Stop repeating the same Jira call in every hook

Say you built three of the hooks from this folder. The one in [prompt-submit.md](prompt-submit.md) fetches a ticket when you mention its number. The one in [tool-describe.md](tool-describe.md) checks a ticket exists. The one in [attribution-text.md](attribution-text.md) reads the ticket off the branch.

All three contain this line:

```ts
const res = await $.mcp.call('atlassian', 'getJiraIssue', { key })
```

Jira renames a field. You now edit three hooks and miss one.

The hook adds `jira` to the engine, once.

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

Every hook now writes `await $.jira.issue('ABC-1')`. When Jira changes, you edit one function.

**Why not a shared function you import?** An imported function is only a function. This one is an event, which is what the next example needs.

**The bill.** Nothing.

**The trap.** You are adding a name to a surface every plugin shares. Pick one nobody else will pick.

---

## 2. Fetch the same ticket once instead of six times

You mention ABC-1 six times in a session. Jira gets called six times. Five of those calls return the same answer and cost you a wait each.

You do not touch the code from example 1. `$.jira.issue` is an event, so a second hook sits on top of it.

```ts
on('jira.issue', async ($, e, next) => {
  const hit = await $.store.get(`jira:${e.key}`)
  if (hit !== undefined) return hit
  const fresh = await next(e)
  await $.store.set(`jira:${e.key}`, fresh)
  return fresh
})
```

The first call goes to Jira. The next five come from `$.store`. Nothing else changed, and the hook that added `$.jira` does not know this happened.

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

## 4. Turn your repository's scripts into engine methods

Your repository has a script that lists open work and one that checks a file. Three of your hooks call them, each spelling out the path and the arguments. Someone moves the scripts into a subfolder and all three break.

The hook puts them behind one name.

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

The hooks call `$.repo.openWork()`. When the script moves, you change one line here.

**Why not call the scripts directly?** Three hooks calling two scripts is six places to update. This is one.

**The bill.** Nothing.

**The trap.** A method that hides a slow script is still slow. Say so in the name or the comment.

---

**Now go argue with this page.** It was written by someone who had the feature for an afternoon. A sketch that turns out wrong is worth more than the page is.
