# ui.press - a button in the terminal

**Nothing here has been run.** These sketches come from the type declarations of a feature that shipped hours ago. Try one, break it, tell everyone what happened.

---

A plugin can draw a `Button`. When somebody presses it, this event fires. **You** are the person at the keyboard. **The hook** is the code you install.

| | |
|---|---|
| The hook receives | `{ plugin, element, component, surface }` |

`plugin` is who drew the button and `element` is which one, written as a path like `0.1.convert`. Another plugin can hook your button by matching on both.

The handler you wrote on the button is not the hook. It is what runs at the bottom of the chain, after every `ui.press` hook has had its turn.

### "Why not just write it in CLAUDE.md?"

A question in the chat needs a turn to ask and a turn to answer. A button needs neither.

---

## 1. Offer the conversion right after the file is written

Claude writes a slide deck. You want a PDF as well. You type a follow-up message, wait for a turn, and get one.

The hook draws a button when that kind of file appears, and does the work when you press it.

```tsx
on('ui.render', { component: 'ToolResult' }, async ($, e, next) => {
  const { Box, Button } = await $.ui.resolve(e)
  const path = (e.props.output as string) ?? ''
  if (!path.endsWith('.pptx')) return next(e)
  return <Box>{await next(e)}<Button key="pdf" label="Convert to PDF" /></Box>
})

on('ui.press', { element: 'pdf' }, async ($, e, next) => {
  await $.process.run(['soffice', '--headless', '--convert-to', 'pdf', lastFile])
  $.ui.toast('PDF written next to the original.')
})
```

No turn, no message, no waiting for a model.

**Why not ask Claude?** Because the conversion is a command, not a decision. A model is not needed to run it.

**The bill.** Nothing until you press it.

**The trap.** The press handler runs outside a turn. Claude does not know it happened. If the result matters to the conversation, say so with `$.prompt.submit`.

---

## 2. Send a draft to review with one press

Claude writes a blog post. You want two subagents to check it: one for accuracy, one for tone. Today you type that request out.

The hook draws two buttons under the file.

```tsx
on('ui.press', { element: 'review-facts' }, async ($, e, next) => {
  await $.agent.spawn({
    subagentType: 'general-purpose',
    description: 'fact check the draft',
    prompt: 'Read drafts/latest.md. List every claim that has no source. Quote each one.',
    background: true,
  })
  $.ui.toast('Fact check started in the background.')
})
```

A plugin's spawn always runs in the background, so you keep working while it runs.

**The bill.** One subagent per press.

**The trap.** Background work reports back on its own schedule. Tell the user it started, or the button feels broken.

---

## 3. Approve or reject without a message

Claude drafts something that needs your decision. You reply "yes, do it", which costs a turn and a re-read of the whole context.

The hook puts the decision on a button.

```tsx
on('ui.press', { element: 'approve' }, async ($, e, next) => {
  await $.fs.writeFile('drafts/latest.approved', 'yes')
  $.ui.toast('Approved. The next turn will pick it up.')
})
```

The decision is recorded as a file. Any later hook or turn can read it.

**Why not answer in chat?** Because "yes" costs a full turn of context, and the answer is one bit.

**The bill.** One file write.

**The trap.** A decision recorded in a file is invisible in the transcript. Write down what was approved as well as that it was.

---

## 4. Change what another plugin's button does

A plugin you installed has a button that deletes something. You want a confirmation first, and you do not want to fork the plugin.

Your hook runs above theirs and can stop the chain.

```tsx
on('ui.press', { plugin: 'cleaner', element: '0.2.delete' }, async ($, e, next) => {
  const answer = await $.ui.ask('This deletes the folder. Sure?', ['No', 'Yes'])
  if (answer !== 'Yes') return
  return next(e) // their handler runs at the bottom of the chain
})
```

**Why not remove the plugin?** Because you want the rest of it.

**The bill.** Nothing.

**The trap.** Element paths change when the other plugin changes its layout. This breaks silently. Check it after their updates.

---

**Now go argue with this page.** It was written by someone who had the feature for an afternoon. A sketch that turns out wrong is worth more than the page is.
