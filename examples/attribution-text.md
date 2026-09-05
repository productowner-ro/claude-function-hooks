# attribution.text - what gets written on your commits

**Nothing here has been run.** These sketches come from the type declarations of a feature that shipped hours ago. Try one, break it, tell everyone what happened.

---

Claude adds text to commits, pull requests and a few other places. This event hands you that text first. **You** are the person at the keyboard. **The hook** is the code you install.

| | |
|---|---|
| The hook receives | `{ kind, text }` |
| `kind` is one of | `commit`, `pr`, `exemption`, `remedy` |

### "Why not just write it in CLAUDE.md?"

A rule about commit messages is followed until the session is long and the commit is urgent. Then it is not.

---

## 1. Put the ticket number on every commit

Six months later somebody asks why a line exists. `git blame` gives them a commit message that describes the change, not the reason for it.

The hook reads the ticket key from the branch name and puts it in the subject.

```ts
on('attribution.text', { kind: 'commit' }, async ($, e, next) => {
  const branch = (await $.process.run(['git', 'branch', '--show-current'])).stdout.trim()
  const key = branch.match(/[A-Z]{2,4}-\d+/)?.[0]
  if (key === undefined || e.text.includes(key)) return next(e)
  return `${key}: ${e.text}`
})
```

Every commit on that branch carries the key. `git log --grep` becomes useful.

**Why not type it?** You do, most of the time. The commits that miss it are the urgent ones you most want to trace.

**The bill.** One subprocess per commit.

**The trap.** A branch with no key gets nothing. Decide whether that should be a warning.

---

## 2. Remove the attribution footer

Your team does not want commits signed by a tool. Right now you edit that out by hand, or you do not notice.

The hook strips it.

```ts
const FOOTERS = /\n*(Co-Authored-By: Claude.*|🤖 Generated with.*|Claude-Session:.*)/g

on('attribution.text', ($, e, next) => next(e).then((t) => t.replace(FOOTERS, '').trimEnd()))
```

**Why not a setting?** There may be one. This works regardless, and it covers pull request bodies as well.

**The bill.** Nothing.

**The trap.** Some teams want the attribution and some licences ask for it. Check before you remove it.

---

## 3. Enforce your commit message format

Your repository uses one format. Claude uses a different one about a third of the time, and your linter rejects it after the fact.

The hook rewrites the subject line.

```ts
const TYPES = ['feat', 'fix', 'docs', 'chore', 'refactor']

on('attribution.text', { kind: 'commit' }, async ($, e, next) => {
  const text = await next(e)
  const [subject, ...rest] = text.split('\n')
  if (TYPES.some((t) => subject.startsWith(`${t}:`) || subject.startsWith(`${t}(`))) return text
  return [`chore: ${subject.replace(/^[a-z]+:\s*/i, '')}`, ...rest].join('\n')
})
```

**Why not the linter?** The linter rejects the commit after it is written. This makes the commit correct.

**The bill.** Nothing.

**The trap.** Guessing the type wrong is worse than having no type. When you cannot tell, use the neutral one, as the sketch does.

---

## 4. Add reviewers to the pull request body

A pull request goes up. The right people find out about it two days later.

The hook looks at which files changed and names the people who own them.

```ts
on('attribution.text', { kind: 'pr' }, async ($, e, next) => {
  const text = await next(e)
  const files = (await $.process.run(['git', 'diff', '--name-only', 'main...HEAD'])).stdout
  const owners = new Set<string>()
  if (files.includes('migrations/')) owners.add('@data-team')
  if (files.includes('infra/')) owners.add('@platform')
  if (owners.size === 0) return text
  return `${text}\n\nSuggested reviewers: ${[...owners].join(', ')}`
})
```

**Why not a CODEOWNERS file?** Use one if you have one. This works when you do not, and it can suggest rather than require.

**The bill.** One subprocess per pull request.

**The trap.** A suggestion in the body is not a request for review. It is a prompt for a human to click the button.

---

**Now go argue with this page.** It was written by someone who had the feature for an afternoon. A sketch that turns out wrong is worth more than the page is.
