# Claude Code function hooks

A function hook is a JavaScript or TypeScript function that wraps one of the engine's own actions. A plugin registers hooks; the engine calls them around a tool call, a prompt, a drawing, or the start of a session. The hook reads the action, changes it, answers it, or lets it through.

Before function hooks a plugin could only run a shell command at a fixed point. A function hook composes: several plugins wrap the same action, one inside the next, and each decides what the ones beneath it see.

Status: early access. Anthropic shipped it in Claude Code 2.1.261. The surface changes between releases. Tracking issue: https://github.com/anthropics/claude-code/issues/91870

Written 05.09.2026 against Claude Code 2.1.261.

## Read this first

Every fact below comes from the type declarations the engine writes for the build you run. Generate them and read them; do not trust this file over them.

```
/plugin-types
```

Type that in a Claude Code session. It writes two files into `.claude/types/`:

- `claude-code.d.ts` - the module `claude-code`, every event, every method on `$`, every element and component, and the input schema of each of this build's built-in tools.
- `claude-code-mcp.d.ts` - the input schema of the MCP tools connected right now.

The header of `claude-code.d.ts` carries a `tsconfig.json` that fits a hooks module. Regenerate after a Claude Code update instead of editing.

## The parts of a plugin

```
my-plugin/
  .claude-plugin/plugin.json     the manifest: name, version, description
  hooks/hooks.json               { "modules": ["./my-hooks.ts"] }
  hooks/my-hooks.ts              the hooks module
  tsconfig.json                  points at .claude/types and hooks/
```

`hooks.json` may keep its old `command`, `prompt`, `agent` and `http` entries beside `modules`. They still run.

The hooks module exports one function:

```ts
import type { Register } from 'claude-code'

export const register: Register = (on, options) => {
  on('tool.call', { tool: 'Bash' }, ($, e, next) => {
    if (e.command === 'rm -rf /') return { deny: 'Blocked by my-plugin.' }
    return next(e)
  })
}
```

`register` runs once, when the plugin loads. It only registers; it does no work. The engine therefore knows which events a plugin hooks before any hook runs.

`options` holds the values of the fields the manifest declares under `userConfig`. Its type is `Readonly<Record<string, string | number | boolean | readonly string[]>>`. A plugin that reads options without declaring `userConfig` gets a warning in the debug log and every field reads as absent.

## The three arguments

Every hook has the same shape: `($, e, next)`.

| Argument | What it is |
|---|---|
| `$` | The engine interface. Everything the hook may read or do. The module has no DOM and no Node, so `$` is the only door. |
| `e` | The event: the action's input, as a plain immutable value. |
| `next` | The continuation. `next(e)` runs the hooks beneath and then the engine's own behaviour, and resolves to the result. |

`next` also carries four fields about this one dispatch:

| Field | What it gives |
|---|---|
| `next.signal` | An `AbortSignal` that fires when the dispatch ends or is abandoned. Stop any work you started for it. |
| `next.event` | The event's name, for a hook registered on `*`. |
| `next.is(name, e)` | A type predicate. Under `*` it narrows `e` to that event's input. |
| `next.origin` | The plugin whose hook raised this dispatch, or `engine`. |

The rule: `$` holds what is about the world, `next` holds what is about this dispatch.

## The five placements

A hook decides where its work sits relative to the action.

| Placement | Code | Use it to |
|---|---|---|
| before | `$.ui.log('...'); return next(e)` | Act, then let the action run. |
| after | `const r = await next(e); ...; return r` | Let the action run, then read or change its result. |
| during | `const p = next(e); ...; return p` | Start the action and work alongside it. |
| instead | `return { deny: '...' }` | Answer for the action. It never runs. |
| modifying | `return next({ ...e, timeout: 30 })` | Change what the action sees. |

`e` is immutable. To change it, pass `next` a copy.

## Order decides authority

The engine folds the registered hooks: `on(X, A)`, `on(X, B)`, `on(X, C)` runs as `A(B(C(engine)))`. A plugin registered earlier wraps more, so it holds more authority. The engine's own behaviour sits at the bottom.

Plugin order is: the plugins an administrator prepends, then the plugins in dependency order, then the plugins an administrator appends. Within one plugin, hooks keep their registration order.

An organization's control is one prepended plugin. It sees every event before every other plugin and every result after, and nothing beneath it can go around it.

## Every event

Twenty events. Each is a method on `$`; calling the method runs that event's hook chain.

### Tools

| Event | Input | Result |
|---|---|---|
| `tool.call` | `{ tool, tool_use_id, ...the tool's arguments }` | `{ result, text, ref }`, or `{ deny: string }` |
| `tool.describe` | `{ tool, description }` | the description the model reads |
| `PreToolUse` | the old settings hook, kept for the settings hooks | a decision |

### The prompt

| Event | Input | Result |
|---|---|---|
| `prompt.submit` | `{ text, attachments?, turnId?, wait, origin }` | `{ text, context?, origin? }` or `{ drop: string }` |
| `prompt.section` | `{ name, text }` - one section of the system prompt | the section's text |
| `prompt.context` | `{ blocks }` - the context blocks of the first user message | the blocks that are sent |

`prompt.submit`'s `origin.kind` is a closed set: `composer`, `bridge`, `sdk`, `task-notification`, and others. Read it to tell the user's own Enter from a plugin's or a schedule's.

`prompt.context`'s blocks from the engine are `claudeMd`, `userEmail`, `attachedProject` and `currentDate`. A plugin may add its own.

### The session and the turn

| Event | Input | When it fires |
|---|---|---|
| `session.start` | `{ cwd, surface, interactive }` | Once, when the session is ready. Awaited before the first prompt. |
| `turn.start` | `{ text, turnId }` | The turn begins. |
| `turn.step` | `{ turnId, index, answer, toolUses, stopReason }` | Each model step inside the turn. |
| `turn.complete` | the turn's outcome | The turn ends. |

### Subagents and skills

| Event | Input |
|---|---|
| `agent.offer` | which subagent types the model may spawn |
| `agent.spawn` | `{ tool_use_id, prompt, description, subagentType, model?, parentModel, background, fork, name?, cwd? }` |
| `skill.prompt` | `{ skill, text }` - a skill's instructions as the model reads them |
| `attribution.text` | `{ kind, text }` where kind is `commit`, `pr`, `exemption` or `remedy` |

### Drawing

| Event | Input | Result |
|---|---|---|
| `ui.render` | `{ surface, component, requestId, props, viewport? }` | the tree to draw |
| `ui.resolve` | the same render input | the element table for that surface |
| `ui.press` | `{ plugin, element, component, surface }` | what the press does |
| `ui.input` | a text field changed | |
| `ui.select` | a select changed | |

### The engine itself

`engine.create` builds `$`. It is the one event not reachable as `$.noun.event`. Its bottom returns the empty table; the core plugin adds the primitives; each other plugin adds its own nouns to what `next(e)` returned.

```ts
on('engine.create', async ($, e, next) => {
  const below = await next(e)
  return { ...below, store: createStore(below) }
})
```

A step adds nouns and may withhold them. It cannot replace one: changing what a method does is a hook on that method. An organization's prepended plugin returns last, so it decides which nouns survive.

## The engine interface

`$` is an object of nouns, each an object of methods. Every method is an event another plugin can hook. A module that reaches outside `$` fails `claude plugin validate`, so a plugin's side effects are readable from its source.

### `$.plugin`

| Member | Type |
|---|---|
| `name` | `string` - from plugin.json; debug lines carry it |
| `root` | `string` - the plugin's directory, absolute |

### `$.ui` - display

| Method | Signature |
|---|---|
| `log` | `(text: string) => void` - one transcript line |
| `notice` | `(tool_use_id: string, text: string \| undefined) => void` - a line under an open dialog |
| `toast` | `(text: string, options?: { timeoutMs?: number }) => void` |
| `status` | `(text: string \| undefined) => void` |
| `ask` | `(question: string, options?: string[] \| { options?, header?, multiSelect? }) => Promise<string>` |
| `resolve` | `(e: RenderInput) => Promise<Elements[surface]>` - the element table |
| `invalidate` | `(event) => void` - redraw, or drop the engine's cached answers |

`invalidate` takes a render event name, or `prompt.section`, `prompt.context`, `tool.describe`.

### `$.model` - the model

| Method | Signature |
|---|---|
| `complete` | `({ model, prompt, system?, maxTokens? }) => Promise<string>` |
| `fork` | `({ prompt }) => Promise<ModelForkReply \| null>` |
| `classify` | `(text, labels, options?) => Promise<string \| undefined>` |

### `$.session` - what is true of this session

| Method | Answers |
|---|---|
| `messages()` | `SessionMessage[]`, each `{ role, text, toolUses, toolResults? }` |
| `cwd()` | the working directory |
| `model()` | the session's model |
| `turnCount()` | how many turns ran |
| `id()` | the session id |
| `repo()` | `SessionRepo \| null` |
| `surface()` | `'terminal' \| 'desktop' \| null` |

These shapes are the engine's own, not the Messages API's. `messages()` gives `{ role, text, toolUses }` rows, never content blocks.

### `$.tool` - tools

| Method | Signature |
|---|---|
| `list` | `() => Promise<ToolInfo[]>` where `ToolInfo` is `{ name, description, mcp }` |
| `call` | the `tool.call` event, under a `tool_use_id` of its own |
| `register` | `({ name, description, inputSchema? }) => Promise<...>` |

### `$.agent`, `$.prompt`, `$.turn`, `$.mcp`

| Method | Signature |
|---|---|
| `$.agent.spawn` | the `agent.spawn` event; a plugin's spawn always runs in the background |
| `$.agent.list` | `() => Promise<AgentInfo[]>` |
| `$.prompt.submit` | hands the session a prompt once it is idle |
| `$.turn.abort` | stops the running turn |
| `$.mcp.call` | `(server, tool, args?) => Promise<McpToolResult>` |

### `$.fs` - files

| Method | Signature |
|---|---|
| `readFile` | `(path) => Promise<string>` |
| `writeFile` | `(path, text) => Promise<void>` |
| `listDir` | `(path?) => Promise<FsEntry[]>` |
| `exists` | `(path) => Promise<boolean>` |
| `stat` | `(path) => Promise<FsStat>` |
| `ancestors` | `({ names, of? }) => Promise<FsAncestor[]>` - reads instruction files up the tree, the way the engine reads CLAUDE.md |

Text only, UTF-8. Paths resolve under the session's working directory and it follows the session's `cd`; anything outside rejects. A read also takes an absolute path under the system temp directory; a write does not.

### `$.store` - values that outlive the session

| Method | Signature |
|---|---|
| `get` | `(key) => Promise<unknown>` |
| `set` | `(key, value) => Promise<void>` - JSON data |
| `delete` | `(key) => Promise<void>` |
| `keys` | `() => Promise<string[]>` |

In the CLI it is a JSON file under `~/.claude/plugins/store/`. On the page it is `localStorage` under the plugin's name.

### `$.clock`, `$.http`, `$.process`, `$.audio`

| Method | Signature |
|---|---|
| `$.clock.now` | `() => number` |
| `$.clock.sleep` | `(ms, options?) => Promise<void>` |
| `$.clock.after` | `(ms, fn) => Timer` - `Timer` is `{ cancel }` |
| `$.clock.every` | `(ms, fn) => Timer` |
| `$.http.fetch` | `(url, init?) => Promise<{ status, ok, headers, text }>` |
| `$.process.run` | `(argv, init?) => Promise<{ exitCode, stdout, stderr }>` - by argv, not a shell string |
| `$.audio.play` | `(clip, options?) => Promise<void>` |
| `$.audio.speak` | `(text, options?) => Promise<SpeakResult>` |

## Matchers

`on` takes an optional matcher between the event and the hook. It is a partial of `e`, matched substructurally: an object matches when each of its keys matches, an array matches when any element matches, anything else matches by equality. An empty array matches nothing.

```ts
on('tool.call', { tool: 'Bash' }, ($, e, next) => { /* e is Bash's input */ })
on('ui.render', { component: 'ToolUse', surface: ['terminal', 'desktop'] }, hook)
```

The engine skips a hook whose matcher does not match, and the matcher narrows `e` for TypeScript.

## Registering a tool the model can call

```ts
on('session.start', async ($, e, next) => {
  await $.tool.register({
    name: 'lookup',
    description: 'Looks a code up in the parts table.',
    inputSchema: { type: 'object', properties: { code: { type: 'string' } }, required: ['code'] },
  })
  return next(e)
})

on('tool.call', { tool: 'mcp__my-plugin__lookup' }, async ($, e, next) => {
  return { result: await find(e.code) }
})
```

The tool is listed as `mcp__<plugin>__<name>`. Register it from `session.start`, which the engine awaits before the first prompt, so the tool is listed by turn one. A call no hook answers fails saying so. Registering the same name again replaces the tool.

## Drawing

A `ui.render` hook receives one component instance.

| Field | What it says |
|---|---|
| `e.component` | which of the twelve components |
| `e.surface` | `terminal` or `desktop` |
| `e.requestId` | which instance: the tool_use_id for a tool row, the message id for a message, the agent id for a spinner |
| `e.props` | the component's props, as plain data |
| `e.viewport` | `{ columns, rows }` in character cells, once the surface has measured |

The twelve components: `AskUserQuestion`, `UserMessage`, `AssistantMessage`, `ToolUse`, `ToolResult`, `ToolGroup`, `Spinner`, `TurnDuration`, `InfoNotice`, `SessionMode`, `PromptHint`, `AbovePrompt`.

Build the tree from the table `$.ui.resolve(e)` returns. The terminal offers `Box`, `Text`, `div`, `span`, `b`, `Button`, `Input`, `Select`, `Link`. The desktop offers the same plus `Svg`. Narrowing `e.surface` narrows the table.

```tsx
on('ui.render', { component: 'ToolUse' }, async ($, e, next) => {
  const { Box, Text } = await $.ui.resolve(e)
  const drawn = await next(e)
  return (
    <Box>
      {drawn}
      <Text>{e.props.toolName}</Text>
    </Box>
  )
})
```

JSX compiles to `h`, which the environment provides as a global along with `Fragment`, `Box`, `Text`, `Button` and `Link`. Set `"jsx": "react", "jsxFactory": "h"` in the tsconfig, and leave `lib` without `dom`: the DOM's own `Text` would shadow the element.

A tree that does not validate is not drawn. The engine draws its own component and writes a debug line starting `ui.render (<Component>): a hook returned a tree that does not validate`.

A width change re-runs every hooked site once the resize settles. A height change alone redraws nothing. `$.ui.invalidate` asks for a redraw when the hook's own state changed.

`Button`, `Input` and `Select` keep their handlers inside the plugin and raise `ui.press`, `ui.input` and `ui.select`. The handler is that dispatch's bottom, so another plugin may act before, after or instead of it. Address one by matcher: `on('ui.press', { plugin: 'explainer', element: '0.1.explain' }, hook)`. Keys reach an element only while it has focus; Esc returns to the prompt.

## Work that outlives one dispatch

A hook runs inside one dispatch with a budget. `next.signal` aborts when the dispatch is abandoned. Anything started for the dispatch stops on it.

Work meant to last longer starts from `session.start` and keeps going with `$.clock.every` and `$.clock.after`. Those timers run until cancelled or until the module reloads. `$.prompt.submit` wakes a quiet session with a prompt. `$.ui.status`, `$.ui.toast` and `$.ui.log` show state without starting a turn. `$.store` keeps values across sessions.

## The failure policy

**The engine fails open.** A hook that throws, overruns its budget, or answers the wrong shape is skipped. The hooks beneath and the engine run in its place, or the hook's last `next` result stands. The failure is reported in the debug log, naming the hook.

For a hook that guards something, this is the failure mode to design against: a broken guard does not block, it disappears. Test the guard, do not assume it.

## Running and debugging one

```
claude --plugin-dir ./my-plugin --debug
claude plugin validate ./my-plugin
```

`--plugin-dir` loads the plugin from disk for that session only. Repeat the flag for several. In an interactive session the folder is watched: saving a file reloads the module, `register` runs again in a fresh environment, and the previous environment's timers are dropped.

`claude plugin validate` reads the manifest and the module's source and prints what the module hooks and what it calls on `$`:

```
  ❯ ./my-hooks.ts hooks: prompt.submit, tool.call{tool=Read}, tool.call
  ❯ ./my-hooks.ts calls: $.fs.readFile, $.ui.log
```

That listing is the quickest check that the engine sees what you meant.

The debug log is where the engine names a module it did not load and why, a hook that threw or overran, a tree it refused, and a result it refused. Grep it by the plugin's name:

```
grep -i my-plugin ~/.claude/debug/<session-id>.txt
```

Useful lines:

- `hooks module <name> loaded (worker, environment 1); events: prompt.submit,tool.call` - the module loaded and these events registered.
- `hooks module <name> prompt.submit settled in 16.3ms (next() included)` - the hook ran.
- `plugin <name>: options requested but its manifest declares no userConfig` - add `userConfig` or stop reading options.

Options for a plugin loaded with `--plugin-dir` come from settings under `pluginConfigs`, keyed by `<name>` or `<name>@inline`.

## A worked example

This plugin makes a JSON file behave like an environment variable. A file named `*.hidden.json` reaches the model with every value replaced by a placeholder. The model puts the placeholder into a tool call; the plugin puts the real value in just before the tool runs. The model never reads the value.

`hooks/hooks.json`:

```json
{ "modules": ["./hidden-json.ts"] }
```

`hooks/hidden-json.ts`:

```ts
import type { Register } from 'claude-code'

const SUFFIX = '.hidden.json'
const PLACEHOLDER = /\{\{hidden:([^#{}]+)#([^{}]*)\}\}/g

function markerOf(file: string, trail: string[]): string {
  return `{{hidden:${file}#${trail.join('.')}}}`
}

function redact(value: unknown, file: string, trail: string[]): unknown {
  if (Array.isArray(value)) {
    return value.map((item, i) => redact(item, file, [...trail, String(i)]))
  }
  if (value !== null && typeof value === 'object') {
    const out: Record<string, unknown> = {}
    for (const [key, item] of Object.entries(value as Record<string, unknown>)) {
      out[key] = redact(item, file, [...trail, key])
    }
    return out
  }
  return markerOf(file, trail)
}

function lookup(root: unknown, trail: string[]): unknown {
  let node: unknown = root
  for (const segment of trail) {
    if (node === null || typeof node !== 'object') return undefined
    node = (node as Record<string, unknown>)[segment]
  }
  return node
}

function fill(value: unknown, values: Map<string, string>): unknown {
  if (typeof value === 'string') {
    return value.replace(PLACEHOLDER, (marker) => values.get(marker) ?? marker)
  }
  if (Array.isArray(value)) return value.map((item) => fill(item, values))
  if (value !== null && typeof value === 'object') {
    const out: Record<string, unknown> = {}
    for (const [key, item] of Object.entries(value as Record<string, unknown>)) {
      out[key] = fill(item, values)
    }
    return out
  }
  return value
}

export const register: Register = (on) => {
  // Answer the Read instead of letting it run.
  on('tool.call', { tool: 'Read' }, async ($, e, next) => {
    if (!e.file_path.endsWith(SUFFIX)) return next(e)
    let parsed: unknown
    try {
      parsed = JSON.parse(await $.fs.readFile(e.file_path))
    } catch (error) {
      return { deny: `hidden-json cannot read ${e.file_path}: ${String(error)}` }
    }
    const content = JSON.stringify(redact(parsed, e.file_path, []), null, 2)
    const numLines = content.split('\n').length
    return {
      result: {
        type: 'text',
        file: { filePath: e.file_path, content, numLines, startLine: 1, totalLines: numLines },
      },
    }
  })

  // Rewrite every other call's input before it runs.
  on('tool.call', async ($, e, next) => {
    const markers = [...JSON.stringify(e).matchAll(PLACEHOLDER)]
    if (markers.length === 0) return next(e)
    const values = new Map<string, string>()
    for (const [marker, file, key] of markers) {
      if (values.has(marker)) continue
      let parsed: unknown
      try {
        parsed = JSON.parse(await $.fs.readFile(file))
      } catch (error) {
        return { deny: `hidden-json cannot read ${file}: ${String(error)}` }
      }
      const value = lookup(parsed, key === '' ? [] : key.split('.'))
      if (value === undefined) return { deny: `hidden-json: ${file} has no key ${key}.` }
      values.set(marker, typeof value === 'string' ? value : JSON.stringify(value))
    }
    return next(fill(e, values) as typeof e)
  })
}
```

Four things this example shows:

1. **Answer instead of `next`.** Hook 1 returns its own `{ result }` in the tool's own output shape. The engine validates it against the tool's output schema, maps it for the model, and records it as the tool's result.
2. **Rewrite the input.** Hook 2 returns `next(rewritten)`. The tool runs with the real value; the transcript keeps the placeholder.
3. **Registration order.** Hook 1 registers first, so it sits above hook 2. A `Read` of a hidden file stops at hook 1 and never reaches hook 2.
4. **No state.** The plugin re-reads the file every time. Nothing is cached between calls, so nothing can go stale or leak.

## What the example does not cover

Worth knowing before you build a guard of this kind, because each is a limit of the event surface, not of the code:

- `Bash` and `Grep` read a file directly. They are `tool.call` events, so a hook can deny them, but nothing redacts them.
- Nothing scans tool results. A tool that echoes a value back prints it to the model.
- A `Write` or an `Edit` that carries a placeholder writes the real value to a plain file. The next read of that file shows it.

## The finding that cost the most time

**An `@file` mention never reaches a hook.** The engine expands it into the first user message before the turn starts. The debug log shows `prompt.submit` running and no `tool.call` dispatching at all, and the file's text is not in `prompt.submit`'s `e.text` either. So neither a `tool.call` hook nor a text scrub can touch it.

Two answers exist at `prompt.submit`:

```ts
// Stop the turn. The text is shown to the user as the reason.
return { drop: 'Name the path without the @.' }

// Or take the @ out of the text and attach your own version as a context block.
const answer = await next({ ...e, text: e.text.replaceAll('@' + file, file) })
return { ...answer, context: [...(answer.context ?? []), myRedactedCopy] }
```

A `prompt.submit` hook that adds context must add to the context its `next` gave it and may not leave an entry out. The whole context is capped like a prompt text.

The general lesson: **check what actually dispatched before you believe a hook covers a path.** The debug log names every event that ran. A hook that registers is not a hook that fires.

## Vocabulary

| Term | Meaning |
|---|---|
| plugin | The unit of installation: a manifest and its skills, agents, MCP servers and hooks. |
| hooks module | The `.js`, `.ts`, `.jsx` or `.tsx` file that `hooks.json` names under `modules`. |
| hook | A function registered on an event. Always `($, e, next)`. |
| event | A method on `$` being called. Calling it runs that event's hook chain. |
| `$` | The engine interface: every affordance a plugin has. |
| surface | Where a drawing goes: `terminal` or `desktop`. Each declares its own elements and components. |
| dispatch | One run of one event's chain. `next.signal` belongs to it. |

## Two rules the engine enforces

- **No re-entry.** A hook is never entered again while its own frame is dispatching. The engine skips it silently and runs every other hook. So a `prompt.submit` hook may call `$.prompt.submit` without looping, and a `*` hook still logs it.
- **The bottom throws.** Below the last hook, `next` throws immediately. There is no existence check to write.
