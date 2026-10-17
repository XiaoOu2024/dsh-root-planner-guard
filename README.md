# dsh-root-planner-guard

> **Archived — this guard now ships inside the preset.**
>
> `dsh-root-planner-preset` (>= 1.3.0) carries this exact module as one of its own preset
> rows (`dsh-root-planner-preset/guard`), so one install gives you the persona, the child
> tool mask and the fence together. This package is **deprecated on npm** and this
> repository is read-only.
>
> That merge was the right call, not just tidiness: the `_agents/` convention is defined by
> the preset's persona. A guard installed on its own fences agents that were never told
> where they may write — it does not protect them, it only stops them.
>
> **Install the preset instead:**
> <https://github.com/XiaoOu2024/dsh-root-planner-preset>

A profile-level DSH plugin that keeps every **delegated subagent** inside the workspace
its parent assigned to it. It is the enforcement half of
[dsh-root-planner-preset](https://github.com/XiaoOu2024/dsh-root-planner-preset), but it
works with any preset: it keys off the session header, not off the persona.

## What it does

For every session whose header says `origin: subagent`, the guard inspects each tool call
before it runs and decides from the call's own arguments.

| Tool | Rule | Strength |
|---|---|---|
| `write`, `edit`, `ops_replace` | The target path must resolve inside `<session cwd>/_agents/` | **Structural** — the call is refused and never executes |
| `pwsh`, `ops_bash` | `workdir` must resolve inside `_agents/`; an absolute path in the command that points at the parent's files is refused | **Best effort** — see the limitation below |
| everything else | untouched | — |

The root session is never fenced: it owns the workspace and has to be able to write
anywhere in it to integrate a child's artifacts.

A refusal reads like this, and states the boundary rather than only refusing:

```
write: outside your workspace. You may write only inside
"D:\work\project\_agents"; everything else in the parent's directory is
read-only to you. Send your parent a change request instead of writing
"D:\work\project\report.md" directly.
```

## The shell limitation, stated plainly

A shell command can name a path no argument scan will see: `$env:HOME` + `'..'`, a script
that writes, a Here-String, a variable holding an absolute path. The guard enforces the
two things it can prove — the working directory is inside the workspace, and no absolute
path written into the command points at the parent's files — and lets anything assembled
at runtime through.

This is not laziness. Confining a child's shell properly needs a narrower sandbox write
root, and DSH cannot give one: `sandboxPolicy.resolve()` accepts only `{ session, mode }`
and derives `workspaceRoot` from `session.header.cwd`, which is immutable and inherited
from the parent; `AgentOptions` carries no `cwd`, and the spawn provider's config has only
`providerName`.

So pair this with a parent-side check. The preset's persona already does:

```sh
touch _agents/.mark-<label>                                   # before delegating
find . -path ./_agents -prune -o -type f -newer _agents/.mark-<label> -print
```

Empty output means the child touched nothing outside its own directory.

## Install

```sh
dsh plugin add dsh-root-planner-guard --registry https://registry.npmjs.org/
```

The `--registry` matters on a machine whose configured registry is a mirror: pnpm's
supply-chain check reads the mirror's manifest, and a lagging mirror reports the version
as missing.

Restart `dsh web` afterwards. A plugin installed while the Harness is running does not
load a fresh module generation, so an edited or newly activated guard does not take
effect until the process restarts.

## Configuration

```yaml
- id: root-planner-guard
  name: dsh-root-planner-guard
  config:
    enabled: true        # false leaves the calls alone without unloading the plugin
    directoryName: _agents
    guardShell: true     # false limits enforcement to the file tools
    debug: false         # true appends one line per fenced call to guard-debug.log
```

The plugin declares no `Config` schema, so the row is validated by this code rather than
by the Loader. That is deliberate: a plain JSON Schema `Config` export makes Cordis throw
`Cannot read properties of undefined (reading 'validate')` at activation, because the
Loader validates a declared Config with a zod schema.

## How it is wired

One guard is registered on the plugin's own context, so it sees every tool call and decides
from the calling agent's session header (`exec.agent.session.header`). It is not one
listener per agent: that design needs the `agent/created` payload shape and a per-agent
context, and it has to tear itself down when a child is disposed. A single global guard
needs none of that.

`tools.guard()` is used because it is synchronous and monotonic: it refuses the call
instead of asking the model not to make it, and no later guard can force-allow a call this
one denied.

## Test

```sh
node test-guard.mjs
```

22 assertions over the boundary logic: containment, relative and absolute paths, `..`
escapes, the `_agents-other` prefix trap, case-insensitivity, missing path arguments, and
the shell rules.

## License

MIT
