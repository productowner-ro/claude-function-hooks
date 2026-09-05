# ui.select - a dropdown a plugin drew

**Nothing here has been run.** These sketches come from the type declarations of a feature that shipped hours ago. Try one, break it, tell everyone what happened.

---

A plugin can draw a `Select`. When somebody picks an option, this event fires. **You** are the person at the keyboard. **The hook** is the code you install.

| | |
|---|---|
| The hook receives | the select and the option chosen |

A select is the right shape when there is a small, known list of answers. A button is for one action. A field is for free text. This is for choosing between things.

### "Why not just write it in CLAUDE.md?"

The choice is yours, not Claude's. This is how you make it without spending a turn.

---

## 1. Switch cluster without leaving the session

You need to point at a different cluster. You leave the terminal, run a command, come back, and forget which one you are on.

The hook draws a select of your contexts and switches when you pick one.

```tsx
on('ui.select', { element: 'cluster' }, async ($, e, next) => {
  const target = e.value
  if (target.includes('prod')) {
    const answer = await $.ui.ask(`Switch to ${target}?`, ['No', 'Yes'])
    if (answer !== 'Yes') return
  }
  await $.process.run(['kubectl', 'config', 'use-context', target])
  $.ui.invalidate('ui.render')
  $.ui.toast(`Now on ${target}`)
})
```

The invalidate call redraws the banner from `ui-render.md`, so the screen and the machine agree.

**Why not the command?** Because the command does not confirm, and it does not update your banner.

**The bill.** One subprocess per switch.

**The trap.** The select changes your real environment. Confirm on the dangerous options, as the sketch does.

---

## 2. Pick who should review, from the people who could

A change is ready. Two teams could review it. You want to choose, not type a name.

The hook offers the candidates it worked out from the changed files.

```tsx
on('ui.select', { element: 'reviewer' }, async ($, e, next) => {
  await $.fs.writeFile('.review-request', e.value)
  $.ui.toast(`${e.value} recorded as reviewer.`)
})
```

The choice is written to a file. The next turn, or a later hook, acts on it.

**Why not decide in chat?** Because a list of four names is a list, not a conversation.

**The bill.** One file write.

**The trap.** Recording the choice is not making the request. Be clear on screen about which one happened.

---

## 3. Change the plugin's behaviour without editing settings

Your plugin has three modes: quiet, normal, verbose. Changing mode means editing a settings file and restarting.

The hook stores the choice.

```tsx
on('ui.select', { element: 'mode' }, async ($, e, next) => {
  await $.store.set('mode', e.value)
  $.ui.invalidate('ui.render')
  $.ui.toast(`Mode: ${e.value}`)
})
```

Every other hook reads `$.store.get('mode')` and behaves accordingly. `$.store` outlives the session, so the choice sticks.

**Why not a settings file?** Because you would have to restart to change your mind.

**The bill.** Nothing.

**The trap.** A setting you cannot see is a setting you will forget. Put the current mode in the status line.

---

## 4. Choose which model does the background work

Some of your hooks call a model. Sometimes you want the cheap one, sometimes the good one, and it depends on the day.

The hook stores the choice for the other hooks to read.

```tsx
on('ui.select', { element: 'helper-model' }, async ($, e, next) => {
  await $.store.set('helper-model', e.value)
  $.ui.toast(`Background work now uses ${e.value}.`)
})
```

The verbosity rewriter, the compactor and the classifier all read that key instead of naming a model.

**The bill.** Nothing.

**The trap.** A model name that no longer exists fails at the point of use, far from where you chose it. Check the name against `$.session.model()` or a known list.

---

**Now go argue with this page.** It was written by someone who had the feature for an afternoon. A sketch that turns out wrong is worth more than the page is.
