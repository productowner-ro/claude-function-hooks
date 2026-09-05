# ui.resolve - the building blocks every plugin draws with

**Nothing here has been run.** These sketches come from the type declarations of a feature that shipped hours ago. Try one, break it, tell everyone what happened.

---

Before a plugin draws anything, it asks for the set of elements available on this screen: `Box`, `Text`, `Button` and the rest. That request is itself an event. **You** are the person at the keyboard. **The hook** is the code you install.

| | |
|---|---|
| The hook receives | the same input as `ui.render` |
| The hook hands back | the table of elements |

Plugins registered earlier sit above plugins registered later. If your hook is above another plugin, the elements you hand back are the elements that plugin draws with. It has no way to reach the real ones.

| Screen | Elements |
|---|---|
| terminal | `Box`, `Text`, `div`, `span`, `b`, `Button`, `Input`, `Select`, `Link` |
| desktop | the same, plus `Svg` |

### "Why not just write it in CLAUDE.md?"

None of this is Claude's decision. It is about what other plugins are able to draw.

---

## 1. Give every plugin the same colours

Three plugins draw on your screen. Each one picked its own colours. The result is loud and hard to read.

The hook wraps `Text` so a missing colour becomes yours.

```tsx
on('ui.resolve', async ($, e, next) => {
  const table = await next(e)
  const Text = table.Text
  return { ...table, Text: (props: any) => Text({ ...props, color: props.color ?? 'gray' }) }
})
```

Every plugin below yours now draws in your palette without knowing it.

**Why not ask the plugin authors?** There are three of them and they do not know about each other.

**The bill.** Nothing.

**The trap.** A plugin that relies on its colour to mean something loses that meaning. Override the default, not the explicit choice, as the sketch does.

---

## 2. Stop plugins drawing buttons during a screen share

You are presenting. A plugin draws an interactive button. Somebody in the meeting asks what it does. The demo is now about the button.

The hook replaces `Button` with plain text while a flag is set.

```tsx
on('ui.resolve', async ($, e, next) => {
  const table = await next(e)
  if ((await $.store.get('demo-mode')) !== true) return table
  const Text = table.Text
  return { ...table, Button: (props: any) => Text({ children: `[${props.label ?? 'action'}]` }) }
})
```

Nothing crashes. The button becomes a label. [agent-offer.md](agent-offer.md) reads the same `demo-mode` key to stop subagents starting mid-presentation.

**The bill.** Nothing.

**The trap.** A plugin whose only path forward is that button now has no path forward. Use this for presentation, not as a permission control.

---

## 3. Enforce a width limit

Your terminal is 80 columns. A plugin draws a table designed for 200 and the layout breaks.

The hook wraps `Box` with a maximum width taken from the real viewport.

```tsx
on('ui.resolve', async ($, e, next) => {
  const table = await next(e)
  const max = e.viewport?.columns ?? 80
  const Box = table.Box
  return { ...table, Box: (props: any) => Box({ ...props, width: Math.min(props.width ?? max, max) }) }
})
```

**Why not ask them to be responsive?** They cannot know your terminal. You do.

**The bill.** Nothing.

**The trap.** Forcing a width can cut content off instead of wrapping it. Test with a narrow terminal before you keep this.

---

## 4. Find out which plugin drew which part of your screen

Something on your screen is wrong. You have four plugins installed and no idea which one is responsible.

The counting pattern from [turn-step.md](turn-step.md), keyed by `next.origin`, which names the plugin that raised the request.

```tsx
on('ui.resolve', async ($, e, next) => {
  const key = `draws:${next.origin}:${e.component}`
  await $.store.set(key, (((await $.store.get(key)) as number) ?? 0) + 1)
  return next(e)
})
```

**The bill.** One store write per resolve, which is often.

**The trap.** Turn it off once you have the answer. This is a diagnostic, not a permanent hook.

---

**Now go argue with this page.** It was written by someone who had the feature for an afternoon. A sketch that turns out wrong is worth more than the page is.
