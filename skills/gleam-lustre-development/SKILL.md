---
name: gleam-lustre-development
description: Guides Claude through idiomatic Lustre frontend development. Use when building SPAs, UI components, or interactive applications. Based on lustre_ui patterns from the official Lustre team.
---

# Gleam Lustre Development Skill

This skill guides Claude Code through **idiomatic Lustre development** following patterns from the official `lustre_ui` library.

## Package Version

- Lustre v5.5.2 (requires Gleam >= 1.13.0)

## Primary Sources

1. **[Lustre Documentation](https://hexdocs.pm/lustre/)** - Official docs
2. **[Lustre UI](https://github.com/lustre-labs/ui)** - Official component library (reference implementation)
3. **[Lustre Examples](https://github.com/lustre-labs/lustre/tree/main/examples)** - Official examples

## Core Architecture: Model-Update-View

Every Lustre application follows the Elm Architecture:

```gleam
import lustre
import lustre/effect.{type Effect}
import lustre/element.{type Element}

// TYPES -----------------------------------------------------------------------

type Model {
  Model(
    count: Int,
    // ... state fields
  )
}

type Msg {
  UserClickedIncrement
  UserClickedDecrement
  ApiReturnedData(Result(Data, Error))
}

// MAIN ------------------------------------------------------------------------

pub fn main() {
  let app = lustre.application(init, update, view)
  let assert Ok(_) = lustre.start(app, "#app", Nil)
  Nil
}

// INIT ------------------------------------------------------------------------

fn init(_flags) -> #(Model, Effect(Msg)) {
  let model = Model(count: 0)
  #(model, effect.none())
}

// UPDATE ----------------------------------------------------------------------

fn update(model: Model, msg: Msg) -> #(Model, Effect(Msg)) {
  case msg {
    UserClickedIncrement -> #(Model(..model, count: model.count + 1), effect.none())
    UserClickedDecrement -> #(Model(..model, count: model.count - 1), effect.none())
    ApiReturnedData(Ok(data)) -> #(Model(..model, data: data), effect.none())
    ApiReturnedData(Error(_)) -> #(model, effect.none())
  }
}

// VIEW ------------------------------------------------------------------------

fn view(model: Model) -> Element(Msg) {
  html.div([], [
    html.button([event.on_click(UserClickedDecrement)], [html.text("-")]),
    html.p([], [html.text(int.to_string(model.count))]),
    html.button([event.on_click(UserClickedIncrement)], [html.text("+")]),
  ])
}
```

### Application Levels

```gleam
// Static HTML only (no interactivity)
lustre.element(html.div([], [html.text("Hello")]))

// Interactive without effects (init/update return Model only)
lustre.simple(init, update, view)

// Full application with effects (init/update return #(Model, Effect))
lustre.application(init, update, view)

// Registrable Web Component
lustre.component(init:, update:, view:, options: [...])
```

## MANDATORY: Message Naming Convention

**Messages MUST use Subject-Verb-Object naming that describes WHAT HAPPENED, not what to do:**

```gleam
// ✅ CORRECT: Describes what happened
type Msg {
  UserClickedSubmit
  UserTypedInField(value: String)
  UserPressedEnter
  UserSelectedOption(id: String)
  UserToggledCheckbox(checked: Bool)
  ApiReturnedUsers(Result(List(User), Error))
  ApiReturnedError(Error)
  ParentSetValue(value: String)
  ParentToggledOpen
  TimerFired
  WindowResized(width: Int, height: Int)
}

// ❌ WRONG: Imperative/command style
type Msg {
  Submit           // What does this mean?
  SetValue(String) // Command, not event
  Toggle           // Too vague
  LoadUsers        // Command, not event
  UpdateField      // Command, not event
}
```

**Prefixes by source:**
- `User...` - User interactions (clicks, typing, etc.)
- `Api...` - HTTP/API responses
- `Parent...` - Props from parent component
- `Timer...` / `Window...` / `Dom...` - Browser events
- `Child...` - Events from child components

## Controlled vs Uncontrolled Props

For components that can have state managed by parent OR internally:

```gleam
/// A prop that can be controlled by the parent or managed internally.
pub type Prop(a) {
  Prop(
    value: a,        // Current value
    controlled: Bool, // Is parent controlling this?
    touched: Bool,    // Has user interacted?
  )
}

pub fn new(value: a) -> Prop(a) {
  Prop(value: value, controlled: False, touched: False)
}

/// Set default value (only if not controlled and not touched)
pub fn default(prop: Prop(a), value: a) -> Prop(a) {
  case prop.controlled || prop.touched {
    True -> prop
    False -> Prop(..prop, value: value)
  }
}

/// Control from parent (always updates)
pub fn control(prop: Prop(a), value: a) -> Prop(a) {
  Prop(..prop, value: value, controlled: True)
}

/// User touched (only updates if not controlled)
pub fn touch(prop: Prop(a), value: a) -> Prop(a) {
  case prop.controlled {
    True -> prop
    False -> Prop(..prop, value: value, touched: True)
  }
}
```

**Usage in update:**

```gleam
fn update(model: Model, msg: Msg) -> #(Model, Effect(Msg)) {
  case msg {
    ParentSetDefaultValue(value) ->
      // Only apply if not controlled and not touched by user
      case model.open.controlled || model.open.touched {
        True -> #(model, effect.none())
        False -> {
          let open = Prop(..model.open, value: value)
          #(Model(..model, open: open), effect.none())
        }
      }
      
    ParentSetValue(value) -> {
      // Controlled: always update
      let open = Prop(..model.open, value: value, controlled: True)
      #(Model(..model, open: open), effect.none())
    }
    
    UserToggledOpen ->
      case model.open.controlled {
        // Controlled: emit event, don't update locally
        True -> #(model, emit_change(!model.open.value))
        // Uncontrolled: update locally AND emit event
        False -> {
          let open = Prop(..model.open, value: !model.open.value, touched: True)
          #(Model(..model, open: open), emit_change(!model.open.value))
        }
      }
  }
}
```

## Web Components (Registrable Components)

For reusable components that need their own state:

```gleam
import lustre
import lustre/component

pub const tag: String = "my-component"

pub fn register() -> Result(Nil, lustre.Error) {
  let comp = lustre.component(init:, update:, view:, options: [
    // Open shadow DOM (optional)
    component.open_shadow_root(),

    // Inject CSS into Shadow DOM
    component.adopt_styles("
      :host { display: block; }
      :host(:state(open)) { border: 1px solid blue; }
    "),

    // React to attribute changes
    component.on_attribute_change("value", fn(value) {
      Ok(ParentSetValue(value))
    }),

    // React to property changes (for complex values)
    component.on_property_change("items", {
      decode.list(decode.string)
      |> decode.map(ParentSetItems)
    }),

    // React to context from ancestors
    component.on_context_change("theme", {
      use theme <- decode.field("theme", decode.string)
      decode.success(ThemeChanged(theme))
    }),
  ])
  
  lustre.register(comp, tag)
}

// Public element function
pub fn element(
  attributes: List(Attribute(msg)),
  children: List(Element(msg)),
) -> Element(msg) {
  element.element(tag, attributes, children)
}
```

## Opaque Types for Public APIs

Encapsulate internal structure:

```gleam
/// An accordion item with heading and collapsible panel.
pub opaque type Item(msg) {
  Item(
    name: String,
    attributes: List(Attribute(msg)),
    heading: Element(msg),
    panel: Panel(msg),
  )
}

/// Create an accordion item.
pub fn item(
  name name: String,
  attributes attributes: List(Attribute(msg)),
  heading heading: Element(msg),
  panel panel: Panel(msg),
) -> Item(msg) {
  Item(name:, attributes:, heading:, panel:)
}
```

## Effects

```gleam
import lustre/effect

// No effect
effect.none()

// Batch multiple effects
effect.batch([effect1, effect2, effect3])

// Custom effect (runs immediately after update)
effect.from(fn(dispatch) {
  // Do something async
  dispatch(SomethingHappened(result))
})

// Run before browser paint (like React's useLayoutEffect)
effect.before_paint(fn(dispatch) {
  // Measure DOM, synchronous layout changes
  dispatch(LayoutMeasured(dimensions))
})

// Run after browser paint (like React's useEffect)
effect.after_paint(fn(dispatch) {
  // Animations, non-urgent side effects
  dispatch(AnimationStarted)
})

// Emit custom event (for components)
event.emit("my-event", json.object([
  #("value", json.string(value)),
]))

// Provide context to descendants
effect.provide("context-name", json.object([
  #("theme", json.string("dark")),
]))
```

## Event Handling

```gleam
import gleam/dynamic/decode
import lustre/event

// Simple handler (no decoding needed)
event.on_click(UserClickedButton)
event.on_input(UserTypedInField)
event.on_check(UserToggledCheckbox)
event.on_submit(UserSubmittedForm)

// Custom event with decoder
pub fn on_change(handler: fn(String) -> msg) -> Attribute(msg) {
  event.on("change", {
    use value <- decode.field("detail", decode.string)
    decode.success(handler(value))
  })
}

// Complex event with multiple fields
pub fn on_item_change(handler: fn(String, Bool) -> msg) -> Attribute(msg) {
  event.on("item:change", {
    use id <- decode.subfield(["detail", "id"], decode.string)
    use open <- decode.subfield(["detail", "open"], decode.bool)
    decode.success(handler(id, open))
  })
}
```

### Event Modifiers (v5+)

```gleam
// Handler with modifiers (preventDefault, stopPropagation, debounce, throttle)
event.advanced("keydown", key_decoder, [
  event.prevent_default,
  event.stop_propagation,
])

// Debounce input (wait N ms after last event)
event.advanced("input", input_decoder, [
  event.debounce(300),
])

// Throttle scroll (at most once per N ms)
event.advanced("scroll", scroll_decoder, [
  event.throttle(100),
])

// Simple handler shortcut (no decoding, just a msg)
event.handler("click", UserClickedThing)
```

## CSS Pseudo-States

For component states (open, selected, disabled, etc.):

```gleam
import lustre/component

// In update function
case model.open {
  True -> component.set_pseudo_state("open")
  False -> component.remove_pseudo_state("open")
}

// CSS can use :state(open) selector
// :host(:state(open)) { ... }
```

## Performance: Memoisation (v5.5.0+)

Memoised rendering skips re-rendering when inputs haven't changed (~60% improvement in benchmarks):

```gleam
import lustre/element

// Value-based memoisation: re-renders only when `data` changes
element.memo(data, fn(data) {
  html.div([], [
    html.text(data.name),
    html.text(data.description),
  ])
})

// Reference-based memoisation: re-renders only when ref changes
let ref = element.ref()
element.ref(ref, fn() {
  expensive_render()
})
```

## Fragments

Group elements without a wrapper node:

```gleam
element.fragment([
  html.h1([], [html.text("Title")]),
  html.p([], [html.text("Content")]),
])
```

## Keyed Rendering for Lists

**ALWAYS use keyed rendering for dynamic lists:**

```gleam
import lustre/element/keyed

fn view_items(items: List(Item)) -> Element(Msg) {
  keyed.element("ul", [], {
    list.map(items, fn(item) {
      #(item.id, html.li([], [html.text(item.name)]))
    })
  })
}
```

## Accessibility (MANDATORY)

### ARIA Attributes

```gleam
import lustre/attribute

// Role
attribute.role("button")
attribute.role("region")

// States
attribute.aria_expanded(is_open)
attribute.aria_pressed(is_pressed)
attribute.aria_disabled(is_disabled)
attribute.aria_controls(panel_id)
attribute.aria_labelledby(heading_id)

// For dynamic content
attribute.aria_live("polite")
attribute.aria_atomic(True)
```

### Keyboard Navigation

```gleam
fn handle_keydown(model: Model) -> Decoder(Handler(Msg)) {
  use key <- decode.field("key", decode.string)
  use target <- decode.field("target", element_decoder())
  
  case key {
    "ArrowDown" -> find_next(model, target)
    "ArrowUp" -> find_previous(model, target)
    "Home" -> find_first(model)
    "End" -> find_last(model)
    "Enter" | " " -> activate(target)
    "Escape" -> close(model)
    _ -> decode.failure(NoOp, "unhandled key")
  }
}
```

### Inert for Hidden Content

```gleam
// Make collapsed content inert (unfocusable, hidden from screen readers)
component.default_slot([
  attribute.inert(model.collapsed)
], children)
```

## File Structure

```
src/
├── app.gleam                    # Main application
├── app/
│   ├── router.gleam             # Routing
│   └── pages/
│       ├── home.gleam
│       └── settings.gleam
└── components/
    ├── ui.gleam                 # Re-exports all components
    └── ui/
        ├── button.gleam         # Simple component (just functions)
        ├── accordion.gleam      # Public API for complex component
        └── accordion/
            ├── root.gleam       # Web component (internal)
            ├── item.gleam       # Web component (internal)
            ├── trigger.gleam    # Web component (internal)
            └── panel.gleam      # Web component (internal)
```

## Simple Components (Functions)

For stateless UI elements, use simple functions:

```gleam
//// Button components.

import lustre/attribute.{type Attribute, class}
import lustre/element.{type Element}
import lustre/element/html
import lustre/event

pub type Variant {
  Primary
  Secondary
  Danger
}

/// Render a button.
pub fn button(
  attributes: List(Attribute(msg)),
  variant: Variant,
  children: List(Element(msg)),
) -> Element(msg) {
  let variant_class = case variant {
    Primary -> "btn-primary"
    Secondary -> "btn-secondary"
    Danger -> "btn-danger"
  }
  
  html.button([class("btn " <> variant_class), ..attributes], children)
}

/// Render a button with click handler.
pub fn button_with_action(
  label: String,
  on_click: msg,
  variant: Variant,
) -> Element(msg) {
  button([event.on_click(on_click)], variant, [html.text(label)])
}
```

## Complex Components (Web Components)

For stateful, interactive components:

```gleam
//// Accordion component following WAI-ARIA patterns.
////
//// ```gleam
//// accordion.view([], [
////   accordion.item(
////     name: "section-1",
////     attributes: [],
////     heading: accordion.heading([], 
////       accordion.trigger([], [html.text("Title")])
////     ),
////     panel: accordion.panel([], [
////       html.p([], [html.text("Content...")])
////     ]),
////   ),
//// ])
//// ```

// ... (see lustre_ui/accordion.gleam for full implementation)
```

## Documentation Standards

Every public function needs:

```gleam
/// The root accordion element.
///
/// #### Attributes
///
/// [`default_value`](#default_value), [`multiple`](#multiple), [`loop`](#loop).
///
/// #### Events
///
/// [`on_value_change`](#on_value_change)
///
/// #### Accessibility notes
///
/// - Home/End moves focus to first/last trigger
/// - Arrow Up/Down navigates between triggers
///
/// #### Managed attributes
///
/// These are set automatically and **must not** be set manually:
///
/// - `role`
/// - `aria-orientation`
///
pub fn view(
  attributes: List(Attribute(msg)),
  children: List(Item(msg)),
) -> Element(msg) {
  // ...
}
```

## OTP Integration (Erlang target, v5.5.0+)

Lustre apps can be supervised as OTP processes:

```gleam
import lustre
import gleam/otp/static_supervisor as supervisor
import gleam/otp/supervision
import gleam/erlang/process

// Named app (for process registry)
let app =
  lustre.application(init, update, view)
  |> lustre.named(process.new_name("my_app"))

// Add to OTP supervision tree
supervisor.new(supervisor.OneForOne)
|> supervisor.add(lustre.supervised(app, initial_args))
|> supervisor.start

// Factory for on-demand component spawning
supervisor.new(supervisor.OneForOne)
|> supervisor.add(lustre.factory(app))
|> supervisor.start

// Send messages to a running runtime
lustre.send(runtime, SomeMessage)
```

## Server Components

Run Lustre apps on the Erlang server, streaming DOM patches to a ~10kb client runtime:

```gleam
import lustre
import lustre/server_component

// Start server-side (Erlang only)
let app = lustre.application(init, update, view)
let assert Ok(runtime) = lustre.start_server_component(app, initial_args)

// Client-side: render the server component element
server_component.element([
  server_component.route("/ws/my-component"),
  server_component.method(server_component.WebSocket),
  // Specify which event properties to serialize to the server
  server_component.include(["target.value", "key"]),
])

// Emit events to connected clients
server_component.emit("notification", json.string("hello"))
```

**Transport methods:** `WebSocket`, `ServerSentEvents`, `Polling`

## Testing with Simulation

Lustre provides testing utilities for snapshot and behavior testing:

```gleam
import lustre/dev/simulate
import lustre/dev/query

// Create and start a simulated app
let app = simulate.application(init, update, view)
let sim = simulate.start(app, Nil)

// Dispatch messages directly
let sim = simulate.message(sim, UserClickedIncrement)
let sim = simulate.message(sim, UserClickedIncrement)
simulate.model(sim).count  // 2

// Simulate DOM events
let sim = simulate.click(sim, on: query.element(matching: query.tag("button")))
let sim = simulate.input(sim, on: query.element(matching: query.id("name")), value: "Alice")

// Query the virtual DOM
let view = simulate.view(sim)
query.find(in: view, matching: query.element(matching: query.class("active")))
query.has(in: view, matching: query.element(matching: query.text("Hello")))

// Inspect event history
simulate.history(sim)  // List of dispatched events
```

## Common Mistakes to Avoid

1. **Imperative message names**: Use `UserClickedSave` not `Save`
2. **Forgetting to map effects**: Use `effect.map(ChildMsg)` when delegating
3. **Forgetting to map elements**: Use `element.map(ChildMsg)` for child views
4. **Non-keyed lists**: Always use `keyed.element` for dynamic lists
5. **Missing accessibility**: Always add ARIA attributes and keyboard support
6. **Hardcoded IDs**: Generate unique IDs with a shortid generator
7. **Blocking the main thread**: Use effects for async operations
8. **Direct DOM manipulation**: Use Lustre's declarative approach

## Development Tools Configuration

Lustre dev tools are configured via the `[tools.lustre]` table in `gleam.toml`. See the [TOML Reference](https://hexdocs.pm/lustre_dev_tools/toml-reference.html) for all configuration options.

### Common Configuration

```toml
[tools.lustre]

# Build configuration
[tools.lustre.build]
minify = true            # Minify output (reduces size)
outdir = "./dist"        # Output directory

# Dev server configuration
[tools.lustre.dev]
host = "localhost"       # Bind to localhost or "0.0.0.0" for network access
port = 1234              # Server port

# Proxy API requests to backend
[[tools.lustre.dev.proxy]]
from = "/api"
to = "http://localhost:3000"

# HTML document structure
[tools.lustre.html]
title = "My App"         # Document title
```

### CLI Commands

**Download binary dependencies:**
```bash
# Download Bun (JavaScript runtime and bundler)
gleam run -m lustre/dev add bun

# Download Tailwind CSS
gleam run -m lustre/dev add tailwind
```

**Build project:**
```bash
# Build using your app's name (from gleam.toml), generates index.html
gleam run -m lustre/dev build

# Build a specific module, generates index.html
gleam run -m lustre/dev build your_app/module

# Build multiple entry points (requires manual index.html)
gleam run -m lustre/dev build your_app/page1 your_app/page2
```

**Start development server:**
```bash
# Start dev server with file watching and hot reload
gleam run -m lustre/dev start
```

Configuration from `gleam.toml` is automatically applied. Command-line flags (if supported) override configuration settings.

### Tailwind CSS v4 Integration

Create a CSS file in `src/` with the same name as your project (from `gleam.toml`):

```css
/* src/my_project.css */
@import "tailwindcss";

/* Your custom styles */
@theme {
  --color-primary: #3b82f6;
  --font-heading: "Inter", sans-serif;
}
```

Lustre dev tools will automatically detect and compile Tailwind v4. No `tailwind.config.js` needed!

### API Proxying

Forward API requests to a backend server during development:

```toml
[[tools.lustre.dev.proxy]]
from = "/api"
to = "http://localhost:3000"

[[tools.lustre.dev.proxy]]
from = "/auth"
to = "http://localhost:4000"
```

Requests to `/api/*` are forwarded to `http://localhost:3000/api/*` while preserving the path.

## Companion Ecosystem

- **[rsvp](https://hexdocs.pm/rsvp/)** - HTTP client (recommended over lustre_http)
- **[modem](https://hexdocs.pm/modem/)** - Client-side routing
- **[plinth](https://hexdocs.pm/plinth/)** - Browser API bindings
- **[formal](https://hexdocs.pm/formal/)** - Form validation
- **[birdie](https://hexdocs.pm/birdie/)** - Snapshot testing
- **[houdini](https://hexdocs.pm/houdini/)** - HTML entity escaping
- **[lustre_dev_tools](https://hexdocs.pm/lustre_dev_tools/)** - Dev server with hot reload
- **[lustre/ui](https://github.com/lustre-labs/ui)** - Official UI component library

## References

- [Lustre Docs](https://hexdocs.pm/lustre/)
- [Lustre Dev Tools CLI](https://hexdocs.pm/lustre_dev_tools/lustre/dev.html)
- [Lustre Dev Tools TOML Reference](https://hexdocs.pm/lustre_dev_tools/toml-reference.html)
- [Lustre UI Source](https://github.com/lustre-labs/ui)
- [WAI-ARIA Patterns](https://www.w3.org/WAI/ARIA/apg/patterns/)
