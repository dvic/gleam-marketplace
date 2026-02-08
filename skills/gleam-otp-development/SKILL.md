---
name: gleam-otp-development
description: Guides Claude through building concurrent, fault-tolerant applications with Gleam OTP. Use when creating actors, supervision trees, or building distributed BEAM applications.
---

# Gleam OTP Development Skill

This skill guides Claude Code through building concurrent, fault-tolerant applications with Gleam OTP.

## Package Versions

- `gleam_otp` v1.2.0 (requires Gleam >= 1.11.0)
- `gleam_erlang` >= 1.0.0

## Primary Sources

1. **[Gleam OTP Documentation](https://hexdocs.pm/gleam_otp/)** - Complete OTP reference
2. **[Actor Documentation](https://hexdocs.pm/gleam_otp/gleam/otp/actor.html)** - Actor API reference
3. **[Static Supervisor Documentation](https://hexdocs.pm/gleam_otp/gleam/otp/static_supervisor.html)** - Static supervisor API
4. **[Factory Supervisor Documentation](https://hexdocs.pm/gleam_otp/gleam/otp/factory_supervisor.html)** - Dynamic supervisor API
5. **[Supervision Documentation](https://hexdocs.pm/gleam_otp/gleam/otp/supervision.html)** - Child specifications
6. **[Gleam OTP: Using Supervisors](https://vpgleam.substack.com/p/gleam-otp-using-supervisors)** - Supervisor tutorial

## Quick Reference

### Core Modules
- `gleam/otp/actor` - Actor processes with type-safe messaging (Builder pattern)
- `gleam/otp/static_supervisor` - Supervision trees with predefined children
- `gleam/otp/factory_supervisor` - Dynamic supervision for runtime-spawned children (v1.2.0+)
- `gleam/otp/supervision` - Shared types: child specs, restart strategies
- `gleam/otp/system` - OTP system debugging and introspection
- `gleam/erlang/process` - Low-level process operations

### Removed in v1.0.0 (DO NOT USE)
- ~~`gleam/otp/supervisor`~~ - Replaced by `static_supervisor`
- ~~`gleam/otp/task`~~ - Removed, use [taskle](https://hexdocs.pm/taskle/) instead

See: [Gleam OTP Modules](https://hexdocs.pm/gleam_otp/)

## Common Workflows

### Creating a New OTP Application

```bash
gleam new my_otp_app
cd my_otp_app
gleam add gleam_otp gleam_erlang
```

### Basic Actor Pattern

Consult the actor documentation for:
- Creating actors: [Actor - new](https://hexdocs.pm/gleam_otp/gleam/otp/actor.html#new)
- Builder pattern: [Actor - on_message](https://hexdocs.pm/gleam_otp/gleam/otp/actor.html#on_message)
- Message handling: [Actor - Next](https://hexdocs.pm/gleam_otp/gleam/otp/actor.html#Next)
- Starting actors: [Actor - start](https://hexdocs.pm/gleam_otp/gleam/otp/actor.html#start)
- Calling actors: [Actor - call](https://hexdocs.pm/gleam_otp/gleam/otp/actor.html#call)

### Supervision Tree Setup

For supervision patterns, see:
- [Static Supervisor - new](https://hexdocs.pm/gleam_otp/gleam/otp/static_supervisor.html#new)
- [Static Supervisor - add](https://hexdocs.pm/gleam_otp/gleam/otp/static_supervisor.html#add)
- [Supervision - worker](https://hexdocs.pm/gleam_otp/gleam/otp/supervision.html#worker)

Example structure:
```
Application Supervisor (static_supervisor)
├── Database Pool Supervisor (static_supervisor)
│   ├── Connection 1
│   ├── Connection 2
│   └── Connection N
├── Web Server Supervisor (static_supervisor)
│   ├── HTTP Listener
│   └── Request Handlers
└── Worker Factory (factory_supervisor)
    └── Dynamic workers spawned at runtime
```

See: [Gleam OTP: Using Supervisors](https://vpgleam.substack.com/p/gleam-otp-using-supervisors)

### Dynamic Child Supervision (v1.2.0+)

For spawning children at runtime, use `factory_supervisor`:
- [Factory Supervisor - worker_child](https://hexdocs.pm/gleam_otp/gleam/otp/factory_supervisor.html#worker_child)
- [Factory Supervisor - start_child](https://hexdocs.pm/gleam_otp/gleam/otp/factory_supervisor.html#start_child)

### One-Off Tasks

The `gleam/otp/task` module was removed in v1.0.0. Use alternatives:
- [Taskle Library](https://hexdocs.pm/taskle/) - Elixir-like Task functionality
- `process.spawn` for simple fire-and-forget operations

## Design Patterns

### GenServer-Style Actor

```gleam
// State, Message types, start function, handle function
```

See complete examples: [Actor Examples](https://hexdocs.pm/gleam_otp/gleam/otp/actor.html)

### Worker Pool

For worker pool implementation patterns:
- Use `static_supervisor` for fixed pools
- Use `factory_supervisor` for dynamic pools

### Event Manager

For pub/sub patterns, consult:
- [Process - send](https://hexdocs.pm/gleam_erlang/gleam/erlang/process.html#send)
- Custom event manager implementations

### Registry Pattern

For process registration and discovery:
- [Process - new_name](https://hexdocs.pm/gleam_erlang/gleam/erlang/process.html#new_name)
- [Process - named_subject](https://hexdocs.pm/gleam_erlang/gleam/erlang/process.html#named_subject)
- [Process - register](https://hexdocs.pm/gleam_erlang/gleam/erlang/process.html#register)

## Supervision Strategies

Choose the right restart strategy:

### OneForOne
Restart only the failed child.
Use for: Independent workers

### OneForAll
Restart all children if one fails.
Use for: Tightly coupled processes

### RestForOne
Restart failed child and all started after it.
Use for: Dependent process chains

See: [Static Supervisor Strategies](https://hexdocs.pm/gleam_otp/gleam/otp/static_supervisor.html#Strategy)

## Testing OTP Applications

### Testing Actors

```gleam
import gleam/otp/actor
import gleam/erlang/process

pub fn actor_test() {
  let assert Ok(started) = start_my_actor()
  let result = actor.call(started.data, waiting: 100, sending: fn(s) { MyMessage(s) })
  let assert expected = result
}
```

### Testing Supervisors

Test supervision behavior:
- Child starts correctly
- Child restarts on crash
- Supervisor respects max restart limits

See: [Testing Guide](../../rules/testing-practices.md)

## Monitoring and Debugging

### Erlang Observer

Monitor your OTP application:
```bash
erl -pa build/dev/erlang/*/ebin
```

Then: `:observer.start()`

### Process Monitoring

Use process monitoring functions:
[Process - monitor](https://hexdocs.pm/gleam_erlang/gleam/erlang/process.html#monitor)

### System Debugging

Use OTP system introspection:
- `system.get_state(pid)` - Inspect actor state
- `system.suspend(pid)` / `system.resume(pid)` - Pause/resume actors

See: [System Module](https://hexdocs.pm/gleam_otp/gleam/otp/system.html)

### Logging

Integrate logging:
[Palabres - OTP Logging](https://hexdocs.pm/palabres/)

## Common Anti-Patterns

Refer to: [OTP Anti-Patterns](../../rules/otp-patterns.md)

Key anti-patterns to avoid:
- Processes as state (use variables instead)
- Unsupervised processes
- Large messages between processes
- Using processes for code organization

## Libraries for OTP Development

### Core
- **[gleam_otp](https://hexdocs.pm/gleam_otp/)** - OTP framework
- **[gleam_erlang](https://hexdocs.pm/gleam_erlang/)** - Erlang runtime

### Extended
- **[taskle](https://hexdocs.pm/taskle/)** - Elixir-like Task functionality
- **[glixir](https://hexdocs.pm/glixir/)** - Safe Gleam-Elixir OTP interop

### Utilities
- **[palabres](https://hexdocs.pm/palabres/)** - Logging for OTP apps

## Hot Code Reloading

Gleam supports Erlang's hot code reloading, but type safety isn't guaranteed during upgrades.

See: [Gleam FAQ - Hot Code Reloading](https://gleam.run/frequently-asked-questions/)

## Deployment Considerations

### Release Building

Use Gleam's Erlang release functionality:
```bash
gleam export erlang-shipment
```

See: [Deploying to Fly.io](https://gleam.run/deployment/fly/)

### Configuration

Use environment variables:
[Envoy Library](https://hexdocs.pm/envoy/)

### Clustering

For distributed Erlang clusters, consult:
- [Gleam Erlang - Node](https://hexdocs.pm/gleam_erlang/gleam/erlang/node.html)
- Erlang distribution documentation

## Example Applications

Find complete OTP application examples:
- [Gleam OTP Examples](https://github.com/gleam-lang/otp/tree/main/examples)
- Community projects using OTP

## When to Use OTP

Use OTP when you need:
- Long-running stateful services
- Fault tolerance and automatic restarts
- Concurrent independent operations
- Process isolation

Don't use OTP for:
- Simple pure computations
- Organizing code (use modules)
- Holding simple state (use variables)

See: [OTP Patterns](../../rules/otp-patterns.md)

---

**Remember**: OTP is powerful but has specific use cases. Consult official documentation for current patterns and best practices.
