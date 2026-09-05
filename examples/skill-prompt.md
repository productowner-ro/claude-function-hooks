# skill.prompt - editing a skill you did not write

**Nothing here has been run.** These sketches come from the type declarations of a feature that shipped hours ago. Try one, break it, tell everyone what happened.

---

A skill is a set of instructions Claude loads when it starts a particular kind of task. This event hands you those instructions before Claude reads them. **You** are the person at the keyboard. **The hook** is the code you install.

| | |
|---|---|
| The hook receives | `{ skill, text }` |
| The hook hands back | the instructions Claude reads |

This is the event that lets you change somebody else's skill without copying it.

### "Why not just write it in CLAUDE.md?"

Your note in `CLAUDE.md` and the skill's own instructions are two voices. When they disagree, the skill usually wins, because it arrives at the moment of the work.

---

## 1. Change one line in an installed skill

You use a skill that asks you six questions at once. You want one question at a time. The skill belongs to somebody else and updates regularly.

The hook appends the change.

```ts
on('skill.prompt', { skill: 'grilling' }, ($, e, next) =>
  next({ ...e, text: `${e.text}\n\nOverride: ask ONE question per round, not the whole set. Wait for the answer before the next one.` }),
)
```

No fork, no copy, no patch to reapply. Their updates keep arriving, your change stays on top.

**Why not copy the skill?** Because then you own it, and you stop getting the improvements.

**The bill.** A few words when that skill runs.

**The trap.** Your override and their instructions can contradict each other. Say plainly that yours wins, as the sketch does.

---

## 2. Add your repository's rules to a general skill

A skill that writes tickets does not know your date format, your project keys, or your definition of done.

The hook attaches them.

```ts
on('skill.prompt', { skill: 'ticket-writer' }, async ($, e, next) => {
  const rules = await $.fs.readFile('docs/ticket-rules.md')
  return next({ ...e, text: `${e.text}\n\n## This repository\n\n${rules}` })
})
```

The rules live in a file your team edits. The skill picks them up without anyone touching it.

**Why not `CLAUDE.md`?** Those rules only matter while a ticket is being written. Here they cost nothing the rest of the time.

**The bill.** The file's words, when that skill runs.

**The trap.** If the file goes missing the hook throws and is skipped, silently. Check `$.fs.exists` first.

---

## 3. Test whether a change to a skill helps

You rewrote a skill's instructions. You think the new version is better. That is a feeling.

The hook serves each version to half your sessions.

```ts
on('skill.prompt', { skill: 'code-review' }, async ($, e, next) => {
  const id = await $.session.id()
  const variant = id.charCodeAt(0) % 2 === 0 ? 'a' : 'b'
  await $.store.set(`skill-variant:${id}`, variant)
  if (variant === 'a') return next(e)
  return next({ ...e, text: await $.fs.readFile('experiments/code-review-b.md') })
})
```

Note which sessions produced reviews you acted on. After twenty sessions you have evidence.

**The bill.** Nothing.

**The trap.** If you do not record the outcome, you have built a random skill picker.

---

## 4. Find out which skills you actually use

You have installed a lot of skills. Some have not run in months.

The hook counts each one.

```ts
on('skill.prompt', async ($, e, next) => {
  const key = `skill-used:${e.skill}`
  await $.store.set(key, (((await $.store.get(key)) as number) ?? 0) + 1)
  return next(e)
})
```

Read the counts with `$.store.keys()` whenever you want to prune.

**The bill.** Nothing.

**The trap.** A skill used twice a year can still be the one that saves a day each time.

---

**Now go argue with this page.** It was written by someone who had the feature for an afternoon. A sketch that turns out wrong is worth more than the page is.
