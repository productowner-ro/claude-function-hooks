# PreToolUse - the permission check, written in code

**Nothing here has been run.** These sketches come from the type declarations of a feature that shipped hours ago. Try one, break it, tell everyone what happened.

---

`PreToolUse` is the older permission event. Shell scripts in your settings file already hook it, and they keep working. **You** are the person at the keyboard. **The hook** is the code you install.

| | |
|---|---|
| The hook receives | the legacy permission event |
| The hook hands back | a decision |

A shell hook gets a string and can exit non-zero. A function hook gets the whole engine. It can ask you a question, read the file, check the clock, or call a small model.

### "Why not just write it in CLAUDE.md?"

Permissions are the one place where asking politely was never the plan.

---

## 1. Ask only when the file matters

You approve every write, so you stop reading them. The one that mattered goes through with the rest.

The hook stays silent on normal files and stops on the ones that hold secrets.

```ts
const SENSITIVE = ['Write', 'Edit', 'Bash']

on('PreToolUse', async ($, e, next) => {
  if (!SENSITIVE.includes(e.tool)) return next(e)
  const path = e.file_path ?? ''
  if (!/\.(env|pem|key)$|secrets|credentials/.test(path)) return next(e)
  const answer = await $.ui.ask(`Write to ${path}?`, ['No', 'Yes, I meant it'])
  return answer.startsWith('Yes') ? next(e) : { deny: 'You blocked it.' }
})
```

**Why not the built-in prompt?** It cannot tell a boring file from an important one. This can.

**The bill.** Nothing until it fires.

**The trap.** A pattern that is too wide brings back the habit of pressing yes.

---

## 2. Block deploys at the wrong time

Nobody should deploy at 11pm on a Friday. Everybody has.

The hook reads the clock and refuses.

```ts
on('PreToolUse', ($, e, next) => {
  const now = new Date($.clock.now())
  const late = now.getHours() >= 19 || now.getDay() === 5
  if (!late || !/deploy|release/.test(e.command ?? '')) return next(e)
  return { deny: 'Deploys are blocked after 19:00 and on Fridays. Ask a human to override.' }
})
```

**The bill.** Nothing.

**The trap.** During a real incident somebody must override this. Build the override, and name it in the refusal message.

---

## 3. Catch a dangerous command you did not think of

You wrote a list of dangerous commands. There are more spellings than your list has entries.

The hook sends the command to a small model and asks what kind of command it is.

```ts
on('PreToolUse', async ($, e, next) => {
  if (e.tool !== 'Bash') return next(e)
  const verdict = await $.model.classify(e.command,
    ['destructive', 'writes-data', 'read-only'], { model: 'claude-haiku-4-5-20251001' })
  if (verdict !== 'destructive') return next(e)
  const answer = await $.ui.ask(`This looks destructive:\n${e.command}`, ['Block', 'Allow'])
  return answer === 'Allow' ? next(e) : { deny: 'Blocked as destructive.' }
})
```

**The bill.** One small model call per shell command. You feel it. Limit it to commands that already look suspicious.

**The trap.** "Read-only" from a model is an opinion. Use this to widen a net, never as the only net.

---

## 4. Move off your shell hooks one rule at a time

You have shell scripts doing this today. They work. You do not want a rewrite weekend.

The hook answers for one command and passes everything else to the old scripts.

```ts
on('PreToolUse', { tool: 'Bash' }, ($, e, next) => {
  if (!/^git push --force/.test(e.command)) return next(e) // the old scripts still decide
  return { deny: 'Force push is blocked. Push a new branch instead.' }
})
```

You migrate at your own pace. Nothing has to break first.

**The bill.** Nothing.

**The trap.** Two systems with overlapping rules will disagree one day. Write down which one owns what.

---

**Now go argue with this page.** It was written by someone who had the feature for an afternoon. A sketch that turns out wrong is worth more than the page is.
