# OTP Patterns and Anti-Patterns

This document outlines patterns and anti-patterns for building OTP applications in Gleam using gleam_otp v1.2.0.

Based on: https://hexdocs.pm/gleam_otp/

## What is OTP?

OTP (Open Telecom Platform) is the BEAM's actor framework for building fault-tolerant, concurrent, multi-core programs. Gleam's Actor is similar to Erlang's `gen_server` and Elixir's `GenServer`, but with a fully typed interface.

## When to Use OTP

Use OTP processes when you need:

1. **Concurrency** - Multiple operations running simultaneously
2. **Isolation** - Failures contained to specific processes
3. **Fault tolerance** - Automatic recovery from crashes
4. **State management** - Long-lived state that survives between operations
5. **Message passing** - Asynchronous communication between components
6. **Supervision** - Automatic restart of failed components

## Actor Pattern (gleam/otp/actor)

Actors are the most common building block of Gleam OTP programs. They hold state, execute code, and communicate by sending and receiving messages.

### Basic Actor Example

```gleam
import gleam/otp/actor
import gleam/erlang/process.{type Subject}

// Define the message type
pub type Message {
  GetCount(reply_with: Subject(Int))
  Increment
  Decrement
}

// Start the actor
pub fn start() -> Result(Subject(Message), actor.StartError) {
  actor.new(0)
  |> actor.on_message(handle_message)
  |> actor.start
}

// Handle each message
fn handle_message(
  count: Int,
  message: Message,
) -> actor.Next(Int, Message) {
  case message {
    GetCount(client) -> {
      process.send(client, count)
      actor.continue(count)
    }
    Increment -> actor.continue(count + 1)
    Decrement -> actor.continue(count - 1)
  }
}
```

### Using the Actor

```gleam
pub fn main() {
  let assert Ok(counter) = start()

  // Send messages
  process.send(counter, Increment)
  process.send(counter, Increment)

  // Synchronous call
  let count = process.call(counter, GetCount, 1000)
  // count == 2
}
```

### Actor with Custom Initialiser

For actors that need setup before handling messages:

```gleam
import gleam/otp/actor

pub fn start() -> Result(Subject(Message), actor.StartError) {
  actor.new_with_initialiser(5000, fn(subject) {
    // Do initialization work here
    let state = load_initial_state()

    actor.initialised(state)
    |> actor.returning(subject)
  })
  |> actor.on_message(handle_message)
  |> actor.start
}
```

**Initialiser timeout**: First argument is milliseconds to wait. If initialiser takes longer, actor fails to start.

### Named Actors

Register actors with names for discovery:

```gleam
import gleam/erlang/process

pub fn start() -> Result(Subject(Message), actor.StartError) {
  let name = process.new_name("my_actor")

  actor.new(initial_state)
  |> actor.named(name)
  |> actor.on_message(handle_message)
  |> actor.start
}

// Later, send to named actor
pub fn send_to_actor(msg: Message) {
  let name = process.new_name("my_actor")
  let subject = process.named(name)
  process.send(subject, msg)
}
```

### Stopping Actors

```gleam
fn handle_message(
  state: State,
  message: Message,
) -> actor.Next(State, Message) {
  case message {
    Shutdown -> {
      // Clean up if needed
      actor.stop()
    }

    InvalidState(reason) -> {
      // Stop with error
      actor.stop_abnormal(reason)
    }

    ProcessWork(data) -> {
      // Continue processing
      actor.continue(state)
    }
  }
}
```

## Static Supervisor Pattern (gleam/otp/static_supervisor)

Supervisors manage child processes and restart them when they fail. Use `static_supervisor` when the number and types of children are specified once.

### Basic Supervisor

```gleam
import gleam/otp/static_supervisor.{type Supervisor} as supervisor
import gleam/otp/supervision
import gleam/otp/actor

pub fn start_supervisor() -> Result(actor.Started(Supervisor), actor.StartError) {
  supervisor.new(supervisor.OneForOne)
  |> supervisor.add(supervision.worker(start_database))
  |> supervisor.add(supervision.worker(start_cache))
  |> supervisor.add(supervision.worker(start_api_server))
  |> supervisor.start
}

fn start_database() -> Result(actor.Started(Database), actor.StartError) {
  // Start database worker
}

fn start_cache() -> Result(actor.Started(Cache), actor.StartError) {
  // Start cache worker
}

fn start_api_server() -> Result(actor.Started(ApiServer), actor.StartError) {
  // Start API server
}
```

### Restart Strategies

```gleam
import gleam/otp/static_supervisor as supervisor

// OneForOne - Only restart failed child
supervisor.new(supervisor.OneForOne)

// OneForAll - Restart all children if one fails
// Use when children are interdependent
supervisor.new(supervisor.OneForAll)

// RestForOne - Restart failed child and all started after it
// Use when children depend on earlier ones
supervisor.new(supervisor.RestForOne)
```

### Restart Tolerance

Prevent infinite restart loops:

```gleam
supervisor.new(supervisor.OneForOne)
|> supervisor.restart_tolerance(intensity: 5, period: 1)
|> supervisor.add(worker1)
|> supervisor.add(worker2)
|> supervisor.start
```

- **intensity**: Max restarts allowed
- **period**: Time period in seconds

If more than `intensity` restarts occur within `period` seconds, supervisor terminates all children and itself.

**Defaults**: intensity=2, period=5

### Child Specifications

Control how children are restarted:

```gleam
import gleam/otp/supervision

// Worker with custom timeout
supervision.worker(start_my_worker)
|> supervision.timeout(10_000)  // 10 second shutdown timeout

// Restart policies
|> supervision.restart(supervision.Permanent)   // Always restart (default)
|> supervision.restart(supervision.Transient)   // Restart only on abnormal exit
|> supervision.restart(supervision.Temporary)   // Never restart

// Significant children (for auto-shutdown)
|> supervision.significant(True)
```

### Nested Supervisors

Supervisors can supervise other supervisors:

```gleam
import gleam/otp/static_supervisor as supervisor
import gleam/otp/supervision

pub fn start_app() -> Result(actor.Started(supervisor.Supervisor), actor.StartError) {
  supervisor.new(supervisor.OneForOne)
  |> supervisor.add(supervision.supervisor(start_database_supervisor))
  |> supervisor.add(supervision.supervisor(start_web_supervisor))
  |> supervisor.start
}

fn start_database_supervisor() -> Result(actor.Started(supervisor.Supervisor), actor.StartError) {
  supervisor.new(supervisor.OneForOne)
  |> supervisor.add(supervision.worker(start_postgres))
  |> supervisor.add(supervision.worker(start_redis))
  |> supervisor.start
}
```

### Using `supervised()` for Supervision Trees

Instead of calling `start()` directly, use `supervised()` to make the supervisor part of a supervision tree:

```gleam
pub fn my_supervisor() -> supervision.ChildSpecification(supervisor.Supervisor) {
  supervisor.new(supervisor.OneForOne)
  |> supervisor.add(worker1)
  |> supervisor.add(worker2)
  |> supervisor.supervised()  // Returns ChildSpecification
}

// Add to parent supervisor
pub fn start_app() {
  supervisor.new(supervisor.OneForOne)
  |> supervisor.add(my_supervisor())  // Add as child
  |> supervisor.start
}
```

## Task Pattern (gleam/otp/task)

Tasks are for one-off concurrent operations:

```gleam
import gleam/otp/task

pub fn fetch_multiple_resources() -> List(Result(Resource, Error)) {
  let tasks = [
    task.async(fn() { fetch_resource(1) }),
    task.async(fn() { fetch_resource(2) }),
    task.async(fn() { fetch_resource(3) }),
  ]

  list.map(tasks, task.await_forever)
}
```

## Worker Pool Pattern

Use supervisors to create worker pools:

```gleam
import gleam/otp/static_supervisor as supervisor
import gleam/otp/supervision
import gleam/list

pub fn start_worker_pool(size: Int) -> Result(actor.Started(supervisor.Supervisor), actor.StartError) {
  let workers = list.range(1, size)
    |> list.map(fn(_) { supervision.worker(start_worker) })

  let spec = supervisor.new(supervisor.OneForOne)

  list.fold(workers, spec, supervisor.add)
  |> supervisor.start
}

fn start_worker() -> Result(actor.Started(Worker), actor.StartError) {
  actor.new(initial_state)
  |> actor.on_message(handle_work)
  |> actor.start
}
```

## OTP Anti-Patterns

### Processes as State (ANTI-PATTERN)

**DON'T** use processes just to hold simple state:

```gleam
// Bad - process just for state
pub type ConfigServer

pub fn start_config() -> Result(Subject(ConfigMessage), actor.StartError) {
  actor.new(load_config())
  |> actor.on_message(handle_config)
  |> actor.start
}

pub fn get_config() -> Config {
  actor.call(config_server, GetConfig, 1000)
}
```

**DO** use regular variables or pass state as arguments:

```gleam
// Good - simple state
pub fn run(config: Config) {
  // Use config directly
  process_request(config)
}
```

Use processes when you need **concurrency, isolation, or fault tolerance**, not just for holding state.

### Unsupervised Processes (ANTI-PATTERN)

**DON'T** start processes without supervision:

```gleam
// Bad - unsupervised
pub fn start_app() {
  let assert Ok(_) = actor.new(state1) |> actor.on_message(handle1) |> actor.start
  let assert Ok(_) = actor.new(state2) |> actor.on_message(handle2) |> actor.start
  // If these crash, they won't restart!
}
```

**DO** use supervisors:

```gleam
// Good - supervised
pub fn start_app() {
  supervisor.new(supervisor.OneForOne)
  |> supervisor.add(supervision.worker(start_worker1))
  |> supervisor.add(supervision.worker(start_worker2))
  |> supervisor.start
}
```

### Large Messages (ANTI-PATTERN)

**DON'T** send large data structures between processes:

```gleam
// Bad - large message (copied on send!)
pub type Message {
  ProcessData(data: List(LargeRecord))  // Expensive copy!
}
```

**DO** send references or identifiers:

```gleam
// Good - small message with reference
pub type Message {
  ProcessDataId(id: String)
}

// Fetch data from shared storage using ID
fn handle_message(state, message) {
  case message {
    ProcessDataId(id) -> {
      let data = fetch_from_storage(id)
      // Process data
      actor.continue(state)
    }
  }
}
```

### Organizing Code with Processes (ANTI-PATTERN)

**DON'T** use processes for code organization:

```gleam
// Bad - process just for organization
pub fn start_math_server() {
  actor.new(Nil)
  |> actor.on_message(fn(state, msg) {
    case msg {
      Add(a, b, reply) -> {
        process.send(reply, a + b)
        actor.continue(state)
      }
      Multiply(a, b, reply) -> {
        process.send(reply, a * b)
        actor.continue(state)
      }
    }
  })
  |> actor.start
}
```

**DO** use modules for organization:

```gleam
// Good - module for organization
pub fn add(a: Int, b: Int) -> Int {
  a + b
}

pub fn multiply(a: Int, b: Int) -> Int {
  a * b
}
```

### Synchronous Calls in Hot Paths (ANTI-PATTERN)

**DON'T** use synchronous calls in performance-critical code:

```gleam
// Bad - blocking call for each item
pub fn process_many(items: List(Item)) {
  list.map(items, fn(item) {
    process.call(processor, Process(item, _), 5000)  // Blocks!
  })
}
```

**DO** use async messages or batch operations:

```gleam
// Good - async messages
pub fn process_many(items: List(Item), callback: Subject(Result)) {
  list.each(items, fn(item) {
    process.send(processor, Process(item, callback))
  })
}

// Or batch operations
pub fn process_many(items: List(Item)) {
  process.call(processor, ProcessBatch(items, _), 10_000)
}
```

### Very Short Timeouts (ANTI-PATTERN)

**DON'T** use very short timeouts:

```gleam
// Bad - too short, fragile
process.call(server, Request, 10)  // 10ms
```

**DO** use reasonable timeouts:

```gleam
// Good - reasonable timeout
process.call(server, Request, 5000)  // 5 seconds

// Or longer for slow operations
process.call(database, ComplexQuery, 30_000)  // 30 seconds
```

### Ignoring Shutdown (ANTI-PATTERN)

**DON'T** ignore shutdown messages:

```gleam
// Bad - no shutdown handling
fn handle_message(state: State, message: Message) {
  case message {
    Work(data) -> {
      process(data)
      actor.continue(state)
    }
    // Shutdown ignored!
  }
}
```

**DO** handle shutdown gracefully:

```gleam
// Good - handles shutdown
pub type Message {
  Work(Data)
  Shutdown
}

fn handle_message(state: State, message: Message) {
  case message {
    Work(data) -> {
      process(data)
      actor.continue(state)
    }
    Shutdown -> {
      cleanup(state)
      actor.stop()
    }
  }
}
```

## Testing OTP Code

### Testing Actors

```gleam
pub fn counter_increments_test() {
  let assert Ok(counter) = start_counter()

  process.send(counter, Increment)
  process.send(counter, Increment)

  let count = process.call(counter, Get, 1000)
  assert count == 2
}
```

### Testing Supervisors

```gleam
pub fn supervisor_restarts_failed_worker_test() {
  let assert Ok(sup) = start_supervisor()

  // Get worker PID
  let assert Ok(worker) = get_worker(sup)
  let worker_pid = worker.pid

  // Kill worker
  process.kill(worker_pid)

  // Wait for restart
  process.sleep(100)

  // Worker should be restarted with different PID
  let assert Ok(new_worker) = get_worker(sup)
  assert new_worker.pid != worker_pid
}
```

## Best Practices

### DO: Wrap Actor APIs

Create clean APIs for actors:

```gleam
// Good - wrapper API
pub opaque type Counter {
  Counter(subject: Subject(Message))
}

pub fn new() -> Result(Counter, actor.StartError) {
  actor.new(0)
  |> actor.on_message(handle_message)
  |> actor.start
  |> result.map(fn(started) { Counter(started.data) })
}

pub fn increment(counter: Counter) -> Nil {
  process.send(counter.subject, Increment)
}

pub fn get_count(counter: Counter) -> Int {
  process.call(counter.subject, Get, 1000)
}
```

### DO: Use Supervision Trees

Structure your application as a supervision tree:

```
Application Supervisor (OneForOne)
├── Database Supervisor (OneForOne)
│   ├── Postgres Connection Pool
│   └── Redis Connection
├── Business Logic Supervisor (RestForOne)
│   ├── Event Store
│   ├── Command Handler
│   └── Query Handler
└── Web Supervisor (OneForOne)
    ├── HTTP Server
    └── WebSocket Manager
```

### DO: Configure Restart Tolerance

Adjust based on your needs:

```gleam
// Tolerant for less critical systems
supervisor.restart_tolerance(intensity: 10, period: 5)

// Strict for critical systems
supervisor.restart_tolerance(intensity: 3, period: 1)
```

### DO: Use Transient Restart for Workers

For workers that can fail normally:

```gleam
supervision.worker(start_worker)
|> supervision.restart(supervision.Transient)
```

Only restarts on abnormal exits, not normal completion.

## External Resources

- [gleam_otp Package](https://hexdocs.pm/gleam_otp/)
- [gleam/otp/actor Documentation](https://hexdocs.pm/gleam_otp/gleam/otp/actor.html)
- [gleam/otp/static_supervisor Documentation](https://hexdocs.pm/gleam_otp/gleam/otp/static_supervisor.html)
- [Erlang OTP Design Principles](https://www.erlang.org/doc/design_principles/des_princ.html)
- [Using Supervisors Tutorial](https://vpgleam.substack.com/p/gleam-otp-using-supervisors)
