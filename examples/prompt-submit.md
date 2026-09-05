# prompt.submit - the moment before Claude reads you

**Nothing here has been run.** These sketches come from the type declarations of a feature that shipped hours ago. Try one, break it, tell everyone what happened.

---

Between you pressing Enter and Claude reading your message, there is a gap. It used to be nothing. Now code can sit in it.

That code is a **hook**. On this page, **you** means the person at the keyboard. **The hook** means the code you install. Two actors, and it matters which one does the work.

| | |
|---|---|
| The hook receives | `{ text, attachments?, turnId?, wait, origin }` |
| The hook hands back | `{ text, context?, origin? }` or `{ drop: "why" }` |

`text` is your message. `context` is extra pages Claude reads beside it.

### "Why not just write it in CLAUDE.md?"

Because a memory file is a *request*. It loads every session, costs tokens every time, and then Claude decides whether to follow it. Usually it does. Sometimes it is turn forty and your rule quietly stops mattering.

A hook is not a request. It is a fact. Write it once. After that it runs for free.

---

## 1. Look up a name before Claude has to ask

You mention a ticket number or a colleague's name. Claude does not know it, so it spends two turns and three tool calls finding out. Tomorrow, in a new session, it does the same again.

The hook looks the name up and attaches the answer to your message.

```ts
on('prompt.submit', async ($, e, next) => {
  const key = e.text.match(/\b[A-Z]{2,4}-\d+\b/)?.[0]
  if (key === undefined) return next(e)
  const issue = await $.mcp.call('atlassian', 'getJiraIssue', { key })
  return next({ ...e, context: [...(e.context ?? []), `${key}: ${issue.summary}`] })
})
```

It is not really about tickets:

- A name appears → the hook attaches one line about that person.
- You say "weather" → the hook attaches the city your phone reports.
- You say "outage" → the hook calls your alerting system, and the answer arrives with your question.

**Why not `people.md`?** That file loads whole, every session, mentions or not. Thirty people is a page you pay for constantly and use twice a month. The hook pays for the one name you said.

**The bill.** One lookup per match, and you wait for it.

**The trap.** A sloppy pattern calls your alerting system when you say hello. Match narrowly, cache in `$.store`.

---

## 2. Turn "tomorrow" into a real date

You write "end of next week". Claude works out the date from whatever it thinks today is. Sometimes that is right. Sometimes a date lands in a document, goes out to real people, and is four days wrong.

The hook resolves the word and puts the exact date next to it.

```ts
const WORDS = /\b(today|tomorrow|yesterday|next week)\b/gi

on('prompt.submit', ($, e, next) => {
  if (!WORDS.test(e.text)) return next(e)
  const day = new Date($.clock.now())
  return next({ ...e, text: e.text.replace(WORDS, (w) => `${w} (${resolve(w, day)})`) })
})
```

You keep writing like a human. Claude reads "tomorrow (06.09.2026)".

**Why not a rule in `CLAUDE.md`?** "Always resolve relative dates" works most of the time. A calendar does not work most of the time. It works.

**The bill.** Nothing. No network, no tokens.

**The trap.** The hook reads its own machine's clock. Make it name the timezone.

---

## 3. Let Claude use a password without reading it

You want Claude to call an API that needs a token. Today you either paste the token into the chat, or you keep Claude away from the work.

Put the token in a file the hook watches. Claude gets a copy with every value replaced by a label. It writes the command it wants, label and all. The hook puts the real value back as the command runs.

```ts
on('prompt.submit', async ($, e, next) => {
  const file = e.text.match(/[\w./-]+\.hidden\.json/)?.[0]
  if (file === undefined) return next(e)
  const parsed = JSON.parse(await $.fs.readFile(file))
  // redact() turns every value into {{hidden:<file>#<key>}}
  return next({ ...e, context: [...(e.context ?? []), JSON.stringify(redact(parsed, file))] })
})

on('tool.call', async ($, e, next) => next(await fill($, e))) // labels back to values
```

The command goes out carrying the real token. The transcript, the logs and the model's memory carry the label.

**Why not "never read the credentials file" in `CLAUDE.md`?** That rule depends on Claude following it. A security review will not accept that. The hook removes the value before Claude can see it, which is a fact you can put in a policy document.

**The bill.** One file read.

**The trap, part one.** `@` mentions slip past. The engine glues the file into your first message before any hook wakes up, and it never lands in `e.text`. The sketch strips the `@` and attaches its own redacted copy. Finding this cost an evening.

**The trap, part two.** A hook that crashes is skipped and the turn continues without it. That is fine for a hook that adds a label. It is dangerous for a hook that hides a secret. Test what yours does when it throws.

---

## 4. Attach the file you always attach

You ask "what am I doing today". The answer needs today's task list. Every session starts with you fetching that file by hand, or running a slash command to do it.

The hook watches for those words and attaches the file.

```ts
on('prompt.submit', async ($, e, next) => {
  const hit = ['what next', 'today', 'my queue'].some((t) => e.text.toLowerCase().includes(t))
  const path = `inbox/${today()}-todo-queue.md`
  if (!hit || !(await $.fs.exists(path))) return next(e)
  return next({ ...e, context: [...(e.context ?? []), await $.fs.readFile(path)] })
})
```

**Why not a slash command?** It works until you forget, and then you get a confident answer built on nothing.

**The bill.** The file's words, every matching turn. Cheap for a list. Expensive for a policy manual.

**The trap.** `$.fs` reads text, inside the session folder only. Anything outside is refused.

---

**Now go argue with this page.** It was written by someone who had the feature for an afternoon. A sketch that turns out wrong is worth more than the page is.
