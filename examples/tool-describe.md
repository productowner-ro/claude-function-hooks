# tool.describe - what Claude reads before it picks a tool

**Nothing here has been run.** These sketches come from the type declarations of a feature that shipped hours ago. Try one, break it, tell everyone what happened.

---

Before Claude picks a tool, it reads a short description of that tool. This event hands you the description first. **You** are the person at the keyboard. **The hook** is the code you install, and it can change the text Claude reads.

| | |
|---|---|
| The hook receives | `{ tool, description }` |
| The hook hands back | the description Claude reads |

The words arrive at the moment of the decision, and only for the tool they concern.

### "Why not just write it in CLAUDE.md?"

`CLAUDE.md` is read once, at the start, with everything else. Forty turns later it is far away. A tool description is read every time Claude considers that tool.

---

## 1. Put the warning on the tool it belongs to

Your team has one parameter that must always be passed. You wrote it down in three files. It still goes missing.

The hook appends the rule to that tool's description.

```ts
on('tool.describe', { tool: 'mcp__metabase__execute_query' }, ($, e, next) =>
  next({ ...e, description: `${e.description}\n\nAlways pass instance: "production". The default is staging and returns different numbers.` }),
)
```

**Why not `CLAUDE.md`?** That file competes with everything else in the session. This does not compete with anything.

**The bill.** A few words, on the tools you change.

**The trap.** This is still a nudge. Claude can still forget. Add a `tool.call` hook that fills the parameter in, and you get both.

---

## 2. Show Claude what a tool costs

One of your tools takes 40 seconds and returns 30,000 characters. Another takes one second. Claude cannot tell them apart, so it picks the slow one.

The hook adds the measured cost to the description.

```ts
on('tool.describe', async ($, e, next) => {
  const stats = (await $.store.get(`cost:${e.tool}`)) as { ms: number; chars: number } | undefined
  if (stats === undefined) return next(e)
  return next({ ...e,
    description: `${e.description}\n\nTypical run: ${Math.round(stats.ms / 1000)}s, ~${stats.chars} characters returned.` })
})
```

A `tool.call` hook times each call and writes the numbers to `$.store`. This one reads them back.

**The bill.** One store read per description.

**The trap.** An average hides a tool that is usually fast and sometimes terrible. Store the worst case too.

---

## 3. Retire a tool without deleting it

You built a better version of a tool. The old one still works, and Claude still picks it.

The hook rewrites the old description to say so.

```ts
on('tool.describe', { tool: 'legacy_search' }, ($, e, next) =>
  next({ ...e, description: `DEPRECATED. Use "search_v2" instead. This one misses anything added after 2025. Only use it if search_v2 fails.` }),
)
```

**Why not delete the tool?** Something you forgot about probably still calls it.

**The bill.** Nothing.

**The trap.** Claude follows the description. A description that is wrong produces confident wrong behaviour.

---

## 4. Teach a third-party tool your house rules

You installed a tool somebody else wrote. Its description is generic. Your project has conventions it knows nothing about.

The hook appends your rules to it.

```ts
on('tool.describe', { tool: 'Bash' }, ($, e, next) =>
  next({ ...e, description: `${e.description}\n\nIn this repository: tests run with "make test", never npm. Never run migrations directly.` }),
)
```

You change nothing in the tool and maintain no patch.

**The bill.** Nothing.

**The trap.** Every word you add is read every time. Two sentences is a nudge. Two paragraphs is a tax.

---

**Now go argue with this page.** It was written by someone who had the feature for an afternoon. A sketch that turns out wrong is worth more than the page is.
