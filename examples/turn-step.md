# turn.step - every reply Claude produces

**Nothing here has been run.** These sketches come from the type declarations of a feature that shipped hours ago. Try one, break it, tell everyone what happened.

---

One turn contains several steps. Claude answers, calls a tool, reads the result, answers again. This event fires after each of those. **You** are the person at the keyboard. **The hook** is the code you install.

| | |
|---|---|
| The hook receives | `{ turnId, index, answer, toolUses, stopReason }` |

`stopReason` is worth knowing: `end_turn`, `max_tokens`, `stop_sequence`, `tool_use`, `pause_turn`, `compaction`, `refusal`, `model_context_window_exceeded`.

### "Why not just write it in CLAUDE.md?"

This event is about what already happened. A rule cannot look backwards.

---

## 1. Shorten Claude's answers for reading

Claude writes 400 words. You want 60. Asking for brevity works for a few turns and then stops working.

The hook sends the finished answer to a small model and stores a short version under the message id. A `ui.render` hook draws that version instead.

```ts
on('turn.step', async ($, e, next) => {
  const out = await next(e)
  if (e.answer.length < 400) return out
  const short = await $.model.complete({
    model: 'claude-haiku-4-5-20251001',
    system: 'Rewrite in at most 60 words. Keep every fact, number and path. No preamble.',
    prompt: e.answer,
  })
  await $.store.set(`short:${e.turnId}:${e.index}`, short)
  $.ui.invalidate('ui.render')
  return out
})
```

The transcript keeps the long version, so Claude's next turn reads its own real words. Only your screen changes. See `ui-render.md` for the other half.

**Why not ask for shorter answers?** Because you have, and it lasts about five turns.

**The bill.** One small model call per long answer. A local model makes it free.

**The trap.** This runs after the answer is finished, so the long version appears first and is replaced. That flicker is the cost of not slowing the answer down.

---

## 2. Count how often a rule is broken

You have a rule about answer length, or format, or tone. You believe it is ignored. You have no numbers.

The hook counts. This is the counting pattern the other pages refer to: read a number out of `$.store`, add one, write it back.

```ts
on('turn.step', async ($, e, next) => {
  const words = e.answer.split(/\s+/).length
  const key = words > 200 ? 'over' : 'under'
  await $.store.set(key, (((await $.store.get(key)) as number) ?? 0) + 1)
  return next(e)
})
```

At the end of the week you have two numbers instead of an argument. Read every key back with `$.store.keys()`.

The same three lines count anything an event carries. Which tools get used (section 4 below). Which skills run, in [skill-prompt.md](skill-prompt.md). Which plugin draws which part of your screen, in [ui-resolve.md](ui-resolve.md).

**Why not judge by feel?** Because you remember the bad ones.

**The bill.** Nothing.

**The trap.** Word count is not the rule. It is a measurement of the rule. Do not start optimising the measurement.

---

## 3. Notice a truncated or refused answer

The answer stops in the middle. It reads like a finished answer, so you act on it.

The hook checks the stop reason and says so on screen.

```ts
on('turn.step', ($, e, next) => {
  if (e.stopReason === 'max_tokens') $.ui.toast('The answer was cut off. Ask it to continue.')
  if (e.stopReason === 'refusal') $.ui.toast('Claude refused this one.')
  if (e.stopReason === 'model_context_window_exceeded') $.ui.toast('Out of context. Start a new session.')
  return next(e)
})
```

**Why not read it yourself?** A cut-off answer often ends on a plausible sentence.

**The bill.** Nothing.

**The trap.** These names come from the current build. Check them against the generated types before you rely on them.

---

## 4. Measure which tools Claude actually uses

You installed twelve tools. You have no idea which ones earn their place, and each one costs tokens in every session.

The hook counts tool use per step.

```ts
on('turn.step', async ($, e, next) => {
  for (const use of e.toolUses ?? []) {
    const key = `used:${use.name}`
    await $.store.set(key, (((await $.store.get(key)) as number) ?? 0) + 1)
  }
  return next(e)
})
```

After a month, the tools with a count of zero are candidates for removal.

**The bill.** Nothing.

**The trap.** A tool used once a quarter can still be the most valuable one you have. Counting tells you what is rare, not what is useless.

---

**Now go argue with this page.** It was written by someone who had the feature for an afternoon. A sketch that turns out wrong is worth more than the page is.
