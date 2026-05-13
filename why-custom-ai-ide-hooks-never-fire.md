# Why Your Custom AI IDE Hooks Never Fire — And the Architectural Problem Behind It

*AI coding assistants are evolving into execution runtimes.
But their hook systems are still designed like editor plugins.*

---

## The Illusion of Extensibility

Modern AI coding environments — Cursor, Claude Code, GitHub Copilot, and others — expose hook systems that appear surprisingly flexible.

You can attach commands before shell execution.
Intercept tool usage.
Monitor file reads.
Run validation logic.
Enforce policies.

At first glance, the model feels extensible enough to support sophisticated runtime workflows.

Until you try to add a new event.

Consider this configuration:

```json
{
  "hooks": {
    "beforeShellExecution": [
      {
        "command": "shell_handler.py",
        "timeout": 90
      }
    ]
  }
}
```

Most platforms happily allow multiple handlers under the same event:

```json
"beforeShellExecution": [
  { "command": "shell_handler.py" },
  { "command": "custom_logger.py" },
  { "command": "policy_guard.py" }
]
```

Everything executes correctly.

So naturally, developers assume this should work too:

```json
"new_beforeShellExecution": [
  { "command": "my_custom_handler.py" }
]
```

It doesn’t.

Not in Cursor.
Not in Claude Code.
Not in Copilot.

The configuration parses successfully.
No validation errors appear.
The hook simply never fires.

Silently.

---

# What’s Actually Happening

The issue is not JSON parsing.

The issue is dispatch architecture.

These platforms internally maintain a fixed registry of supported hook event names:

* `beforeShellExecution`
* `beforeMCPExecution`
* `beforeReadFile`
* `preToolUse`
* `sessionStart`

The runtime only dispatches events that exist in this registry.

Conceptually, the execution model looks something like this:

```python
if event_name == "beforeShellExecution":
    run_hooks(...)
elif event_name == "preToolUse":
    run_hooks(...)
```

Your custom heading:

```json
"new_beforeShellExecution"
```

is not invalid.

It is simply unreachable.

The host has no dispatch path for it.

---

# Why Existing Hooks Work

This explains a behavior many developers notice immediately:

Adding extra handlers under an existing event works perfectly.

Example:

```json
"beforeMCPExecution": [
  { "command": "mcp_handler.py" },
  { "command": "audit_logger.py" }
]
```

Both execute.

Because the dispatcher already knows how to invoke `beforeMCPExecution`.

Once the event fires, every registered handler under that event executes in order.

So these systems support:

* Multiple handlers
* Multiple validators
* Multiple deny paths
* Chained execution

…but only for known event types.

---

# The “Router Hook” Workaround

Eventually teams attempt to bypass this limitation using a dispatch router.

The pattern looks like this:

```json
"beforeShellExecution": [
  {
    "command": "user_hook_router.py beforeShellExecution default_handler.py"
  }
],

"new_beforeShellExecution": [
  {
    "command": "user_command.py"
  }
]
```

The idea is straightforward:

1. Intercept an existing event
2. Route internally
3. Trigger user-defined pseudo-events manually

Technically, this works.

Architecturally, it becomes messy very quickly.

---

# Why This Becomes Unmaintainable

## 1. Every Host Event Needs a Router

If users want custom override support for:

* `beforeShellExecution`
* `beforeMCPExecution`
* `preToolUse`

…then every one of those events requires its own routing layer.

That means the platform installer now has to inject dispatch logic into every supported lifecycle event.

The complexity compounds immediately:

* More bootstrap code
* More lifecycle management
* More failure points
* More upgrade coordination

The hook system stops being declarative and starts behaving like middleware infrastructure.

---

## 2. Every Vendor Uses Different Hook Schemas

This is where things become particularly fragile.

Cursor, Claude Code, and Copilot do not share a unified hook specification.

### Cursor

```json
"beforeShellExecution": [...]
```

### Claude Code

```json
{
  "match": "...",
  "hooks": [...]
}
```

### Copilot

Uses another variation entirely.

Now every dispatch router has to be implemented multiple times.

Every vendor update becomes a compatibility problem.
Every schema evolution becomes maintenance overhead.

Instead of building reusable runtime extensions, teams end up building adapter layers.

---

# The Hidden Security Problem

This is where the architecture becomes more than inconvenient.

In most runtime-governed systems, the default handler is not “just another hook.”

It is often responsible for:

* Policy enforcement
* Runtime event publishing
* Audit logging
* Permission validation
* Security contracts
* Deny/allow decisions

If a user replaces that handler with a custom router:

```json
"beforeShellExecution": [
  { "command": "user_router.py" }
]
```

then the user hook becomes the sole gatekeeper for that lifecycle event.

Now several guarantees disappear:

* Centralized auditability
* Consistent policy enforcement
* Reliable runtime telemetry
* Uniform security behavior

Unless the platform forces user-defined hooks to preserve and re-publish the internal governance contract correctly.

Which introduces another enforcement problem entirely.

---

# The Core Architectural Flaw

The deeper issue is this:

> Current AI IDEs treat hooks as static configuration, not as extensible runtime events.

That design works for:

* Simple automation
* Lightweight integrations
* Vendor-controlled extensions
* Local developer scripting

But it breaks down for:

* Runtime governance
* Enterprise observability
* Multi-agent orchestration
* Security middleware
* Policy chaining
* Extensible execution workflows

Because AI coding assistants are no longer behaving like editors.

They are behaving like execution environments.

And execution environments eventually require proper event systems.

---

# What a Better Model Looks Like

Instead of hardcoded dispatch names:

```python
KNOWN_EVENTS = [...]
```

platforms need dynamic event registration.

Something closer to:

```json
{
  "registerEvent": {
    "name": "new_beforeShellExecution",
    "parent": "beforeShellExecution"
  }
}
```

Now hooks become composable rather than hardwired.

---

# Runtime Middleware, Not Plugin Callbacks

Security-critical handlers should also become mandatory middleware layers.

The execution model should look more like:

```text
Core Policy Layer
    ↓
Audit Middleware
    ↓
User Extensions
    ↓
Optional Overrides
```

Not:

```text
User Hook Replaces Everything
```

Because governance systems should be extensible — not bypassable.

---

# The Bigger Industry Shift

This hook limitation reveals a broader transition happening across AI tooling.

We are moving from:

> AI-assisted editors

to:

> AI runtime platforms

And runtime platforms eventually need:

* Middleware systems
* Lifecycle orchestration
* Event buses
* Policy chains
* Observability layers
* Runtime governance

Today’s AI IDE hook systems are still first-generation implementations.

Good enough for lightweight scripting.

Not yet designed for runtime-grade extensibility.

---

# Final Thought

The most dangerous failures in infrastructure are rarely the visible ones.

A hook that crashes loudly gets fixed.

A hook that parses successfully, gets ignored silently, and never executes?

That becomes a governance blind spot.

And as AI coding assistants evolve into autonomous execution environments, those blind spots stop being tooling bugs.

They become infrastructure risks.

