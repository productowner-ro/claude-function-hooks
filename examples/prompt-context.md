# prompt.context - the pages attached to your first message

**Nothing here has been run.** These sketches come from the type declarations of a feature that shipped hours ago. Try one, break it, tell everyone what happened.

---

Claude's first message arrives with blocks attached: your `CLAUDE.md`, your email address, today's date. This event hands you that list before it is sent. **You** are the person at the keyboard. **The hook** is the code you install, and it can add blocks, change them, or remove them.

| | |
|---|---|
| The hook receives | `{ blocks: PromptContextBlock[] }` |
| The hook hands back | the blocks that get sent |

Each block is `{ name, text }`. The engine's own are `claudeMd`, `userEmail`, `attachedProject` and `currentDate`.

### "Why not just write it in CLAUDE.md?"

`CLAUDE.md` is one of these blocks, and it is static. These blocks are built fresh at the start of every session, so they can carry facts that were true five seconds ago.

---

## 1. Tell Claude where your code stands right now

You ask about a bug. Claude does not know which branch you are on, whether you have uncommitted work, or when the last deploy went out. It guesses, or it spends three tool calls finding out.

The hook runs the git commands once and attaches the answer.

```ts
on('prompt.context', async ($, e, next) => {
  const branch = await $.process.run(['git', 'branch', '--show-current'])
  const dirty = await $.process.run(['git', 'status', '--porcelain'])
  const text = `Branch: ${branch.stdout.trim()}\nUncommitted files: ${dirty.stdout.split('\n').filter(Boolean).length}`
  return next({ ...e, blocks: [...e.blocks, { name: 'repoState', text }] })
})
```

Use this for facts Claude should reason with. Use [session-start.md](session-start.md) for facts you need to see, such as a warning that a repository is stale. That event can show a toast; this one cannot.

**Why not let Claude run `git status`?** It costs a tool call and a turn, every session, and only happens if Claude thinks to do it.

**The bill.** Two subprocesses at session start.

**The trap.** This is a photograph, not a live feed. It is taken once and never updated. Say so inside the block, or Claude will still be quoting your branch name an hour after you switched.

---

## 2. Attach the state of your work

You open a session to plan the day. Claude has no idea what you were doing yesterday, what is blocked, or what is waiting on someone else.

The hook reads your task file and attaches a summary.

```ts
on('prompt.context', async ($, e, next) => {
  if (!(await $.fs.exists('tasks/open.md'))) return next(e)
  const text = await $.fs.readFile('tasks/open.md')
  return next({ ...e, blocks: [...e.blocks, { name: 'openWork', text }] })
})
```

Turn one already knows the board.

This event fires once per session, so the file arrives whether or not you ask about it. [prompt-submit.md](prompt-submit.md) attaches the same file only when your words match, which is cheaper and later. Use this one when every session needs it. Use that one when only some do.

**Why not paste it?** You would have to remember, and you would paste a slightly different version every time.

**The bill.** The file's words, in every session. Keep the file short, or summarise it in the hook.

**The trap.** A file that grows quietly makes every session more expensive. Check the size and truncate.

---

## 3. Write the date in the format your team uses

The engine attaches today's date. Your documents use a different format, and mixed formats in a document are a real problem when two countries read it.

The hook rewrites that one block.

```ts
on('prompt.context', ($, e, next) => {
  const blocks = e.blocks.map((b) =>
    b.name === 'currentDate' ? { ...b, text: `Today is ${format(new Date($.clock.now()))}. Write every date this way.` } : b,
  )
  return next({ ...e, blocks })
})
```

This sets the format once per session. It does not resolve the word "tomorrow" in your message. [prompt-submit.md](prompt-submit.md) does that. The two work together: this one says how a date should look, that one says which date you meant.

**Why not a rule?** A rule asks Claude to convert. This gives it the format it should copy.

**The bill.** Nothing.

**The trap.** You changed a block the engine owns. If a future version renames it, your hook silently does nothing. Log when the block is missing.

---

## 4. Remove a block you do not want sent

The engine attaches your email address. On a shared machine, or in a recorded demo, you do not want that.

The hook drops it.

```ts
on('prompt.context', ($, e, next) =>
  next({ ...e, blocks: e.blocks.filter((b) => b.name !== 'userEmail') }),
)
```

**Why not change a setting?** There may not be one. This works whether or not there is.

**The bill.** Nothing.

**The trap.** Something may depend on that block. Remove it, work a day, and see what breaks.

---

**Now go argue with this page.** It was written by someone who had the feature for an afternoon. A sketch that turns out wrong is worth more than the page is.
