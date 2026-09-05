# prompt.section - editing Claude's standing instructions

**Nothing here has been run.** These sketches come from the type declarations of a feature that shipped hours ago. Try one, break it, tell everyone what happened.

---

Claude's system prompt is built from named sections: tone, tool guidance, house rules. Each section passes through this event. **You** are the person at the keyboard. **The hook** is the code you install, and it can rewrite a section, replace it, or remove it.

| | |
|---|---|
| The hook receives | `{ name, text: string \| null }` |
| The hook hands back | the text of that section |

Other events add context. This one changes the instructions Claude works under.

### "Why not just write it in CLAUDE.md?"

`CLAUDE.md` is one of these sections. This event decides what the sections say.

---

## 1. A different tone per folder

You want short technical answers in your code repository and plain prose in your notes folder. Today you get one setting for both.

The hook reads the working directory and swaps the tone section.

```ts
on('prompt.section', { name: 'output-style' }, async ($, e, next) => {
  const cwd = await $.session.cwd()
  if (cwd.includes('/notes')) return 'Write plainly. Short sentences. No code unless asked.'
  if (cwd.includes('/api')) return 'Be terse and technical. Show types. Skip the explanation.'
  return next(e)
})
```

**Why not two settings files?** You would have to remember which one you are in, and switching is a chore you will skip.

**The bill.** Nothing.

**The trap.** A silent change of behaviour is confusing later. Have the hook announce itself once with `$.ui.log`.

---

## 2. Send the reminder when it is needed

Rules loaded at turn one are strongest at turn one. By turn forty the conversation has drifted and the rules are far behind it.

The hook counts the turns and adds a line once the session gets long.

```ts
on('prompt.section', { name: 'house-rules' }, async ($, e, next) => {
  const turns = await $.session.turnCount()
  if (turns < 25) return next(e)
  return `${e.text ?? ''}\n\nThis conversation is long. Re-read the rules above. If the task has changed shape, say so and suggest starting fresh.`
})
```

You stop noticing the drift five turns after it starts.

**Why not say it yourself?** By the time you notice, you already spent those five turns.

**The bill.** A few words, and only late in a session.

**The trap.** Adding text to a long conversation makes it longer. Keep the reminder short.

---

## 3. Remove a section you never use

One section costs tokens in every session and applies to none of your work.

The hook returns `null` for it outside the folder where it matters.

```ts
on('prompt.section', { name: 'notebook-guidance' }, async ($, e, next) => {
  const cwd = await $.session.cwd()
  return cwd.includes('/data') ? next(e) : null
})
```

**Why not ask for it to be removed?** It is relevant to somebody. It is not relevant to this folder.

**The bill.** Negative. That is the point.

**The trap.** Removing guidance removes the behaviour it produced. You may only notice as a vague drop in quality. Change one section, work a week, then judge.

---

## 4. Find out whether a rule does anything

Your rules file only grows, because nobody knows which rules carry weight.

The hook serves the long version to half your sessions and a short version to the other half, and records which session got which.

```ts
on('prompt.section', { name: 'house-rules' }, async ($, e, next) => {
  const id = await $.session.id()
  const variant = id.charCodeAt(0) % 2 === 0 ? 'full' : 'short'
  await $.store.set(`variant:${id}`, variant)
  return variant === 'full' ? next(e) : 'Be accurate. Ask when unsure.'
})
```

After two weeks you compare the work you kept against the variant that produced it.

**Why not just decide?** You have been deciding for a year and the file only gets longer.

**The bill.** Nothing.

**The trap.** You have to record the outcome. Without that, this is a coin flip.

---

**Now go argue with this page.** It was written by someone who had the feature for an afternoon. A sketch that turns out wrong is worth more than the page is.
