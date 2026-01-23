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
   import gleam/otp/supervisor
   import gleam/otp/actor

   pub fn start() -> Result(Subject(supervisor.Message), actor.StartError) {
     supervisor.start(init)
   }

   fn init(children: supervisor.Children) -> supervisor.Children {
     children
     |> supervisor.add(supervisor.worker(start_worker_1))
     |> supervisor.add(supervisor.worker(start_worker_2))
   }
   ```

2. **Create worker actors**
   ```gleam
   // src/worker.gleam
   import gleam/otp/actor

   pub type Message {
     DoWork(reply_with: Subject(Result(Data, Error)))
     Shutdown
   }

   pub fn start() -> Result(actor.Started(Subject(Message)), actor.StartError) {
     actor.new(initial_state())
     |> actor.on_message(handle_message)
     |> actor.start()
   }

   fn handle_message(state: State, message: Message) -> actor.Next(State, Message) {
     case message {
       DoWork(client) -> {
         let result = perform_work(state)
         actor.send(client, result)
         actor.continue(state)
       }
       Shutdown -> actor.stop()
     }
   }
   ```

3. **Configure restart strategy**
   ```gleam
   supervisor.start_spec(supervisor.Spec(
     argument: Nil,
     init: init,
     max_frequency: 5,      // Max 5 restarts
     frequency_period: 1,    // Within 1 second
   ))
   ```

4. **Start supervisor in main**
   ```gleam
   pub fn main() {
     let assert Ok(supervisor) = my_supervisor.start()
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
supervisor.add(children, supervisor.worker(start_worker))
```

### Nested Supervisors
Create supervision trees:
```gleam
fn init(children) {
  children
  |> supervisor.add(supervisor.supervisor(database_supervisor))
  |> supervisor.add(supervisor.supervisor(web_supervisor))
  |> supervisor.add(supervisor.worker(start_logger))
}
```

## Worker Specifications

### Simple Worker
```gleam
supervisor.worker(fn(_arg) {
  start_my_actor()
})
```

### Worker with Arguments
```gleam
supervisor.worker(fn(arg) {
  start_my_actor(arg.config)
})
```

## Restart Limits

Configure how many restarts are allowed:

```gleam
supervisor.Spec(
  max_frequency: 10,      // Maximum 10 restarts
  frequency_period: 60,   // Within 60 seconds
  // If exceeded, supervisor itself crashes
)
```

## Common Patterns

### Database Connection Pool
```gleam
fn init_database_pool(children) {
  list.range(1, pool_size)
  |> list.fold(children, fn(children, _) {
    supervisor.add(children, supervisor.worker(start_connection))
  })
}
```

### Worker Pool with Router
```gleam
pub fn start_pool(size: Int) {
  supervisor.start(fn(children) {
    // Add router
    children
    |> supervisor.add(supervisor.worker(start_router))
    // Add workers
    |> add_workers(size)
  })
}
```

## Monitoring and Debugging

### Check Supervisor Status
Use Erlang Observer to view supervision tree:
```bash
erl -pa build/dev/erlang/*/ebin
```

Then: `:observer.start()`

### Logging
```gleam
import gleam/logging

fn handle_message(message, state) {
  logging.log(logging.Info, "Processing message")
  // Handle message
}
```

## References

- [Gleam OTP - Supervisor](https://hexdocs.pm/gleam_otp/gleam/otp/supervisor.html)
- [OTP Patterns](../rules/otp-patterns.md)
- [OTP Development Skill](../skills/gleam-otp-development/SKILL.md)
- [Using Supervisors Tutorial](https://vpgleam.substack.com/p/gleam-otp-using-supervisors)

## Best Practices

- Always supervise your processes
- Choose appropriate restart strategies
- Configure restart limits
- Use nested supervisors for complex apps
- Test supervision behavior
- Monitor with Observer
- Log important events

## Anti-Patterns to Avoid

- ❌ Unsupervised processes
- ❌ Using processes just for state
- ❌ Large messages between processes
- ❌ Using processes for code organization

See: [OTP Anti-Patterns](../rules/otp-patterns.md)
