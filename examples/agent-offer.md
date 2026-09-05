# agent.offer - which helpers Claude is allowed to see

**Nothing here has been run.** These sketches come from the type declarations of a feature that shipped hours ago. Try one, break it, tell everyone what happened.

---

Claude can start subagents: separate helpers that do a piece of work and report back. This event decides which types it is offered. **You** are the person at the keyboard. **The hook** is the code you install.

| | |
|---|---|
| The hook receives | the list of subagent types Claude may spawn |
| The hook hands back | the list it actually sees |

An agent Claude cannot see is an agent it cannot pick.

### "Why not just write it in CLAUDE.md?"

"Do not use the search agent for operational questions" is a sentence Claude reads at turn one and weighs against everything else at turn thirty. Removing the agent from the list is not a judgement call.

---

## 1. Take the search agent away for operations work

You ask "why is the queue backed up". Claude starts a broad search agent, which reads forty files, costs three minutes and finds a runbook you could have named in one line.

The hook removes the exploration agents when the question is operational.

```ts
const BROAD = ['Explore', 'general-purpose']

on('agent.offer', ($, e, next) => {
  const ops = /\b(queue|pod|deploy|cluster|consumer|incident)\b/i.test(e.text ?? '')
  if (!ops) return next(e)
  return next({ ...e, agents: e.agents.filter((a) => !BROAD.includes(a.name)) })
})
```

Claude does the targeted lookup instead, because the wide option is not there.

**Why not the rule?** Because the rule already exists in most repositories, and the broad search still happens.

**The bill.** Nothing.

**The trap.** Sometimes the broad search is right. Have the hook log what it removed, so you can tell when it removed the wrong thing.

---

## 2. Offer only the agents that fit the folder

You have nine agent types. Six of them make no sense in the repository you are in. Claude still sees all nine and sometimes picks a wrong one.

The hook filters by working directory.

```ts
const BY_FOLDER: Record<string, string[]> = {
  '/docs': ['writer', 'editor'],
  '/api': ['test-runner', 'code-review'],
}

on('agent.offer', async ($, e, next) => {
  const cwd = await $.session.cwd()
  const allowed = Object.entries(BY_FOLDER).find(([k]) => cwd.includes(k))?.[1]
  if (allowed === undefined) return next(e)
  return next({ ...e, agents: e.agents.filter((a) => allowed.includes(a.name)) })
})
```

Fewer options, fewer wrong picks, and a shorter list to describe in every session.

**The bill.** Negative. Each agent you remove was costing tokens to describe.

**The trap.** A folder you forgot to list gets the full set. That is the safe default, but check it is what you meant.

---

## 3. Turn agents off entirely while you are demonstrating

You are screen sharing. A subagent starts, produces three minutes of output nobody can follow, and the point of the demo is lost.

The hook empties the list.

```ts
on('agent.offer', async ($, e, next) => {
  const off = (await $.store.get('demo-mode')) === true
  return off ? next({ ...e, agents: [] }) : next(e)
})
```

Set `demo-mode` from a button, a command, or by hand before you share your screen. [ui-resolve.md](ui-resolve.md) reads the same key to stop plugins drawing buttons during a demo. One switch, two effects.

**Why not close the plugin?** Because you want the rest of it running.

**The bill.** Nothing.

**The trap.** You will forget to turn it off. Put the state in the status line.

---

## 4. Limit new team members to a safe set

Someone joins and starts using agents that write files and run commands, before they know what the repository does.

The hook offers read-only agents until a marker file exists.

```ts
on('agent.offer', async ($, e, next) => {
  if (await $.fs.exists('.trusted')) return next(e)
  const safe = e.agents.filter((a) => ['Explore', 'Plan'].includes(a.name))
  return next({ ...e, agents: safe })
})
```

**Why not tell them?** Telling them works. This works on the day they forget.

**The bill.** One file check.

**The trap.** This is a guide rail, not a security control. Anyone can create the file.

---

**Now go argue with this page.** It was written by someone who had the feature for an afternoon. A sketch that turns out wrong is worth more than the page is.
