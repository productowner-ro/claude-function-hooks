# ui.render - changing what you see, not what Claude reads

**Nothing here has been run.** These sketches come from the type declarations of a feature that shipped hours ago. Try one, break it, tell everyone what happened.

---

Everything on your screen is a component being drawn. This event fires for each one, before it is drawn. **You** are the person at the keyboard. **The hook** is the code you install.

The important fact: this event changes the picture only. The transcript keeps the original, and Claude's next turn reads its own real words. Your screen and Claude's memory can differ on purpose.

| | |
|---|---|
| The hook receives | `{ surface, component, requestId, props, viewport? }` |
| The hook hands back | a tree of elements, or `next(e)` to leave it alone |

The components: `AssistantMessage`, `UserMessage`, `ToolUse`, `ToolResult`, `ToolGroup`, `AskUserQuestion`, `Spinner`, `TurnDuration`, `InfoNotice`, `SessionMode`, `PromptHint`, `AbovePrompt`.

Get your elements with `$.ui.resolve(e)`. On the terminal: `Box`, `Text`, `Button`, `Input`, `Select`, `Link` and a few more.

### "Why not just write it in CLAUDE.md?"

You cannot ask for a badge, a banner or a folded block. Those are drawings.

---

## 1. Read a short version of a long answer

Claude writes 400 words. You want 60. Asking for shorter answers works for a few turns.

A `turn.step` hook stores a short version. This hook draws it.

```tsx
on('ui.render', { component: 'AssistantMessage' }, async ($, e, next) => {
  const { Text } = await $.ui.resolve(e)
  const short = (await $.store.get(`short:${e.requestId}`)) as string | undefined
  if (short === undefined) return next(e)
  return <Text>{short}</Text>
})
```

You read 60 words. Claude reads its own 400 on the next turn, so nothing about its reasoning breaks. See `turn-step.md` for the half that produces the short version.

**Why not ask for brevity?** Because you have, and it lapses.

**The bill.** Whatever the shortening cost. A local model makes it free.

**The trap.** A tree that does not validate is not drawn at all. The engine draws its own version and writes a line to the debug log starting `ui.render (AssistantMessage): a hook returned a tree that does not validate`. Run with `claude --debug` while you build this.

---

## 2. Keep the production warning on screen

You have three terminals open. Two point at staging. One points at production. They look identical.

The hook draws the target in the session mode line.

```tsx
on('ui.render', { component: 'SessionMode' }, async ($, e, next) => {
  const { Box, Text } = await $.ui.resolve(e)
  const ctx = (await $.process.run(['kubectl', 'config', 'current-context'])).stdout.trim()
  const prod = ctx.includes('prod')
  return (
    <Box>
      {await next(e)}
      <Text color={prod ? 'red' : 'green'}>{prod ? ` PRODUCTION (${ctx}) ` : ` ${ctx} `}</Text>
    </Box>
  )
})
```

The warning is on screen before you type the command, not after you run it.

**The bill.** One subprocess per redraw. Cache it and refresh with `$.ui.invalidate('ui.render')`.

**The trap.** Render results are cached. If the cluster changes and you do not invalidate, the banner lies. A lying banner is worse than none.

---

## 3. Fold tool output you were never going to read

A command prints 300 lines. They fill your screen and push the answer out of view.

The hook draws a summary line instead.

```tsx
on('ui.render', { component: 'ToolResult' }, async ($, e, next) => {
  const { Text } = await $.ui.resolve(e)
  const lines = String(e.props.output ?? '').split('\n')
  if (lines.length < 40 || e.props.errored) return next(e)
  return <Text dimColor>{`${e.props.toolName}: ${lines.length} lines, ${lines[0].slice(0, 60)}...`}</Text>
})
```

Claude still receives all 300 lines. Only the drawing changes.

**The bill.** Nothing.

**The trap.** Errors must never be folded. The sketch checks `errored` first. Keep that check.

---

## 4. Put a small dashboard above the prompt

You want to know how long you have been working, how many commits are unpushed, and whether the build is broken. None of that is on screen.

The hook draws it in the strip above the prompt.

```tsx
on('ui.render', { component: 'AbovePrompt' }, async ($, e, next) => {
  const { Box, Text } = await $.ui.resolve(e)
  const started = ((await $.store.get('day-started')) as number) ?? $.clock.now()
  const hours = ((($.clock.now() - started) / 3600000)).toFixed(1)
  const ci = (await $.store.get('ci')) as string | undefined
  return (
    <Box>
      <Text>{`${hours}h today`}</Text>
      {Number(hours) > 8 ? <Text color="yellow"> - long day, take a break</Text> : null}
      {ci !== undefined && ci !== '0' ? <Text color="red">{` ${ci} builds failing`}</Text> : null}
    </Box>
  )
})
```

**Why not another window?** Because you are looking at this one.

**The bill.** Nothing, if the values come from `$.store`.

**The trap.** `e.viewport.columns` tells you the width. A dashboard wider than the terminal breaks the layout. A width change re-runs this hook once the resize settles.

---

## 5. Show a file's git status next to every path

Claude reads a file. You cannot tell whether that file has uncommitted changes.

The hook adds a marker to the tool row.

```tsx
on('ui.render', { component: 'ToolUse' }, async ($, e, next) => {
  const { Box, Text } = await $.ui.resolve(e)
  const path = (e.props.input as { file_path?: string }).file_path
  if (path === undefined) return next(e)
  const status = (await $.process.run(['git', 'status', '--porcelain', path])).stdout.trim()
  if (status === '') return next(e)
  return <Box>{await next(e)}<Text color="yellow"> modified</Text></Box>
})
```

**The bill.** One subprocess per file row. That adds up on a busy turn. Cache by path.

**The trap.** Drawing happens often. Anything slow in a render hook makes the whole display feel slow.

---

**Now go argue with this page.** It was written by someone who had the feature for an afternoon. A sketch that turns out wrong is worth more than the page is.
