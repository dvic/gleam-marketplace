# /gleam-supervisor - Create OTP Supervisor

Create a supervisor for fault-tolerant concurrent applications.

## Usage

```
/gleam-supervisor <supervisor-name>
```

## Workflow

1. **Create supervisor module**
   ```gleam
   // src/<name>_supervisor.gleam
   import gleam/otp/static_supervisor.{type Supervisor} as supervisor
   import gleam/otp/supervision
   import gleam/otp/actor

   pub fn start() -> Result(actor.Started(Supervisor), actor.StartError) {
     supervisor.new(supervisor.OneForOne)
     |> supervisor.add(supervision.worker(start_worker_1))
     |> supervisor.add(supervision.worker(start_worker_2))
     |> supervisor.start
   }

   /// Create a child specification for nesting in another supervisor
   pub fn supervised() -> supervision.ChildSpecification(Supervisor) {
     supervisor.new(supervisor.OneForOne)
     |> supervisor.add(supervision.worker(start_worker_1))
     |> supervisor.add(supervision.worker(start_worker_2))
     |> supervisor.supervised
   }
   ```

2. **Create worker actors**
   ```gleam
   // src/worker.gleam
   import gleam/otp/actor
   import gleam/erlang/process.{type Subject}

   pub type Message {
     DoWork(reply_with: Subject(Result(Data, Error)))
     Shutdown
   }

   pub fn start() -> Result(actor.Started(Subject(Message)), actor.StartError) {
     actor.new(initial_state())
     |> actor.on_message(handle_message)
     |> actor.start
   }

   fn handle_message(state: State, message: Message) -> actor.Next(State, Message) {
     case message {
       DoWork(client) -> {
         let result = perform_work(state)
         process.send(client, result)
         actor.continue(state)
       }
       Shutdown -> actor.stop()
     }
   }
   ```

3. **Configure restart strategy**
   ```gleam
   supervisor.new(supervisor.OneForOne)
   |> supervisor.restart_tolerance(intensity: 5, period: 1)
   |> supervisor.add(supervision.worker(start_worker_1))
   |> supervisor.add(supervision.worker(start_worker_2))
   |> supervisor.start
   ```

4. **Start supervisor in main**
   ```gleam
   pub fn main() {
     let assert Ok(_supervisor) = my_supervisor.start()
     process.sleep_forever()
   }
   ```

5. **Test supervision**
   ```gleam
   pub fn supervisor_starts_children_test() {
     let assert Ok(_supervisor) = my_supervisor.start()
     // Verify children are running
   }
   ```

## Supervision Strategies

### OneForOne
Restart only the failed child:
```gleam
supervisor.new(supervisor.OneForOne)
```

### OneForAll
Restart all children if one fails:
```gleam
supervisor.new(supervisor.OneForAll)
```

### RestForOne
Restart failed child and all started after it:
```gleam
supervisor.new(supervisor.RestForOne)
```

### Nested Supervisors
Create supervision trees:
```gleam
supervisor.new(supervisor.OneForOne)
|> supervisor.add(supervision.supervisor(start_database_supervisor))
|> supervisor.add(supervision.supervisor(start_web_supervisor))
|> supervisor.add(supervision.worker(start_logger))
|> supervisor.start
```

## Child Specifications

### Worker Child
```gleam
supervision.worker(start_my_actor)
```

### Supervisor Child
```gleam
supervision.supervisor(start_sub_supervisor)
```

### Configuring Child Specs
```gleam
supervision.worker(start_my_worker)
|> supervision.timeout(10_000)                   // 10 second shutdown timeout
|> supervision.restart(supervision.Permanent)     // Always restart (default)
|> supervision.restart(supervision.Transient)     // Restart only on abnormal exit
|> supervision.restart(supervision.Temporary)     // Never restart
|> supervision.significant(True)                  // For auto-shutdown
```

## Restart Tolerance

Configure how many restarts are allowed:

```gleam
supervisor.new(supervisor.OneForOne)
|> supervisor.restart_tolerance(intensity: 10, period: 60)
// If exceeded, supervisor itself shuts down
```

Defaults: intensity=2, period=5

## Auto-Shutdown

```gleam
supervisor.new(supervisor.OneForOne)
|> supervisor.auto_shutdown(supervisor.AnySignificant)
// Shuts down when any significant child exits normally
```

Options: `Never` (default), `AnySignificant`, `AllSignificant`

## Factory Supervisor (Dynamic Children)

For spawning children at runtime, use `factory_supervisor`:

```gleam
import gleam/otp/factory_supervisor as factory
import gleam/erlang/process

let workers_name = process.new_name("workers")

let builder =
  factory.worker_child(fn(arg) { start_worker(arg) })
  |> factory.named(workers_name)

let assert Ok(_) = factory.start(builder)

// Later, spawn children dynamically
let supervisor = factory.get_by_name(workers_name)
let assert Ok(started) = factory.start_child(supervisor, worker_arg)
```

## Common Patterns

### Database Connection Pool
```gleam
import gleam/list

pub fn start_database_pool(pool_size: Int) {
  let workers = list.range(1, pool_size)
    |> list.map(fn(_) { supervision.worker(start_connection) })

  let spec = supervisor.new(supervisor.OneForOne)

  list.fold(workers, spec, supervisor.add)
  |> supervisor.start
}
```

## Monitoring and Debugging

### Check Supervisor Status
Use Erlang Observer to view supervision tree:
```bash
erl -pa build/dev/erlang/*/ebin
```

Then: `:observer.start()`

## References

- [Static Supervisor Docs](https://hexdocs.pm/gleam_otp/gleam/otp/static_supervisor.html)
- [Factory Supervisor Docs](https://hexdocs.pm/gleam_otp/gleam/otp/factory_supervisor.html)
- [Supervision Docs](https://hexdocs.pm/gleam_otp/gleam/otp/supervision.html)
- [OTP Patterns](../rules/otp-patterns.md)
- [OTP Development Skill](../skills/gleam-otp-development/SKILL.md)
- [Using Supervisors Tutorial](https://vpgleam.substack.com/p/gleam-otp-using-supervisors)

## Best Practices

- Always supervise your processes
- Choose appropriate restart strategies
- Configure restart tolerance
- Use nested supervisors for complex apps
- Use `supervised()` for composable supervisor trees
- Use `factory_supervisor` for dynamic child pools
- Test supervision behavior
- Monitor with Observer

## Anti-Patterns to Avoid

- Unsupervised processes
- Using processes just for state
- Large messages between processes
- Using processes for code organization

See: [OTP Anti-Patterns](../rules/otp-patterns.md)
