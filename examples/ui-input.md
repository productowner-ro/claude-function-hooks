# ui.input - a text field a plugin drew

**Nothing here has been run.** These sketches come from the type declarations of a feature that shipped hours ago. Try one, break it, tell everyone what happened.

---

A plugin can draw an `Input`. Every time somebody types in it, this event fires with the current text. **You** are the person at the keyboard. **The hook** is the code you install.

| | |
|---|---|
| The hook receives | the field and its new text |

Keys only reach the field while it has focus. Escape returns you to the normal prompt.

This event fires on every keystroke. That is the whole design constraint.

### "Why not just write it in CLAUDE.md?"

Typing into a field is not something you can ask a model to arrange.

---

## 1. Search for a ticket without leaving the terminal

You need a ticket number. You switch to the browser, search, copy the key, come back.

The hook draws a field, searches as you type, and shows the matches.

```tsx
on('ui.input', { element: 'ticket-search' }, async ($, e, next) => {
  if (e.text.length < 3) return next(e)
  const found = await $.mcp.call('atlassian', 'search', { query: e.text, limit: 5 })
  await $.store.set('ticket-matches', found)
  $.ui.invalidate('ui.render')
  return next(e)
})
```

A `ui.render` hook draws the matches from `$.store` under the field.

**The bill.** One search per keystroke, unless you slow it down. Do slow it down.

**The trap.** Firing a network call on every letter is how you get rate limited in an afternoon. Wait until typing pauses, using `$.clock.after` and cancelling the previous timer.

---

## 2. Warn while the text is still being typed

You are writing a message that will go out to people. You notice the problem after you send it.

The hook checks as you type.

```tsx
on('ui.input', { element: 'note' }, async ($, e, next) => {
  const bad = ['obviously', 'just', 'as discussed']
  const hit = bad.find((w) => e.text.toLowerCase().includes(w))
  $.ui.status(hit !== undefined ? `careful: "${hit}"` : undefined)
  return next(e)
})
```

The warning is in the status line. It does not block you and it costs no tokens.

**The bill.** Nothing. Local string checks only.

**The trap.** A warning that fires constantly stops being a warning. Keep the list very short.

---

## 3. Save a draft so it survives

You type three paragraphs into a field, press Escape by accident, and it is gone.

The hook saves every change.

```tsx
on('ui.input', { element: 'note' }, async ($, e, next) => {
  await $.store.set('draft:note', e.text)
  return next(e)
})
```

Read it back with `$.store.get('draft:note')` when you draw the field again, and it comes back filled in.

**The bill.** One store write per keystroke. Fine locally.

**The trap.** Drafts pile up. Delete the key when the text is used.

---

## 4. Find out whether anyone uses the field you built

You added a field to your plugin. You do not know whether it gets used.

The counting pattern from [turn-step.md](turn-step.md), keyed by element. Note that it stores a count and never the text. Do the same.

```tsx
on('ui.input', async ($, e, next) => {
  const key = `input-used:${e.element}`
  await $.store.set(key, (((await $.store.get(key)) as number) ?? 0) + 1)
  return next(e)
})
```

**The trap.** Keystrokes are not uses. Ten letters is one search. Count on submit instead, or divide.

---

**Now go argue with this page.** It was written by someone who had the feature for an afternoon. A sketch that turns out wrong is worth more than the page is.
