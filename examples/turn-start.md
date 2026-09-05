# turn.start - every time Claude begins working

**Nothing here has been run.** These sketches come from the type declarations of a feature that shipped hours ago. Try one, break it, tell everyone what happened.

---

A turn starts when Claude begins working on your message and ends when it stops. This event fires at the beginning of each one. **You** are the person at the keyboard. **The hook** is the code you install.

| | |
|---|---|
| The hook receives | `{ text, turnId }` |

This is the counting event. It fires once per turn, in order, with an id you can use to tie other events together.

### "Why not just write it in CLAUDE.md?"

Counting is not something you can ask for. It is something you do.

---

## 1. Warn before the conversation gets too long

At turn fifty the session gets summarised, and the summary loses detail you needed. You find out when an answer goes strange.

The hook counts turns and warns you before that happens.

```ts
on('turn.start', async ($, e, next) => {
  const n = (await $.session.turnCount()) + 1
  if (n === 40) $.ui.toast('40 turns. Consider finishing this task and starting fresh.')
  if (n > 40) $.ui.status(`turn ${n}`)
  return next(e)
})
```

The warning goes to a toast and the status line. It does not enter the conversation, so it costs no tokens.

**Why not watch it yourself?** The number is not on screen, and you are busy.

**The bill.** Nothing.

**The trap.** One toast is a warning. A toast every turn is noise you will learn to ignore.

---

## 2. Keep a record of your working day

At the end of the week you have to say what you worked on. You reconstruct it from memory and commit messages.

The hook writes one line per turn to a dated file.

```ts
on('turn.start', async ($, e, next) => {
  const day = new Date($.clock.now()).toISOString().slice(0, 10)
  const time = new Date($.clock.now()).toTimeString().slice(0, 5)
  const first = e.text.split('\n')[0].slice(0, 120)
  await $.fs.writeFile(`logs/${day}.md`, `- ${time} ${first}\n`)
  return next(e)
})
```

**Why not the transcript?** The transcript holds everything. This holds what you asked for, one line each, in a file you can read in thirty seconds.

**The bill.** One file append per turn.

**The trap.** `$.fs.writeFile` replaces the file. Read it first and append, or you keep only the last line.

---

## 3. Notice when you are repeating yourself

You ask the same question three times in different words, because the answer keeps missing the point. Nobody notices the pattern, including you.

The hook compares this message to the last few.

```ts
on('turn.start', async ($, e, next) => {
  const recent = ((await $.store.get('recent')) as string[]) ?? []
  const similar = recent.filter((t) => overlap(t, e.text) > 0.6).length
  if (similar >= 2) $.ui.toast('Third time asking something like this. Try a different approach.')
  await $.store.set('recent', [...recent, e.text].slice(-5))
  return next(e)
})
```

**Why not notice yourself?** Because by the third attempt you are annoyed and not counting.

**The bill.** Nothing.

**The trap.** Word overlap is a rough measure. Set the threshold high, or it will fire on every follow-up question.

---

## 4. Time each turn

You have a feeling that some kinds of work take much longer than others. You do not have numbers.

The hook stores the start time here, and a `turn.complete` hook reads it back.

```ts
on('turn.start', async ($, e, next) => {
  await $.store.set(`started:${e.turnId}`, $.clock.now())
  return next(e)
})
```

Pair it with the `turn.complete` sketch that writes the duration out.

**The bill.** Nothing.

**The trap.** Keys accumulate. Delete each one after you read it, or `$.store` fills with dead turn ids.

---

**Now go argue with this page.** It was written by someone who had the feature for an afternoon. A sketch that turns out wrong is worth more than the page is.
