# /gleam:lustre-component - Create Lustre Component

Create a Lustre UI component following idiomatic patterns from lustre_ui.

## Usage

```
/gleam:lustre-component <component_name> [--stateful]
```

## Examples

```
/gleam:lustre-component button
/gleam:lustre-component accordion --stateful
/gleam:lustre-component modal --stateful
```

## Workflow

### Simple Component (Stateless Functions)

For UI elements without internal state, create simple view functions:

**Create `src/components/ui/button.gleam`:**

```gleam
//// Button components with various styles.
////
//// ## Examples
////
//// ```gleam
//// import components/ui/button
////
//// button.primary([event.on_click(UserClickedSubmit)], [html.text("Submit")])
//// button.secondary([event.on_click(UserClickedCancel)], [html.text("Cancel")])
//// ```

import lustre/attribute.{type Attribute, class}
import lustre/element.{type Element}
import lustre/element/html

// TYPES -----------------------------------------------------------------------

/// Button visual variants.
pub type Variant {
  Primary
  Secondary
  Danger
  Ghost
}

// ELEMENTS --------------------------------------------------------------------

/// Render a button with the given variant.
pub fn button(
  attributes: List(Attribute(msg)),
  variant: Variant,
  children: List(Element(msg)),
) -> Element(msg) {
  let base = "px-4 py-2 rounded-md font-medium transition-colors focus:outline-none focus:ring-2"
  
  let variant_classes = case variant {
    Primary -> "bg-black text-white hover:bg-gray-800 focus:ring-gray-500"
    Secondary -> "bg-white border border-gray-300 hover:bg-gray-50 focus:ring-gray-300"
    Danger -> "bg-red-600 text-white hover:bg-red-700 focus:ring-red-500"
    Ghost -> "bg-transparent hover:bg-gray-100 focus:ring-gray-300"
  }
  
  html.button(
    [class(base <> " " <> variant_classes), attribute.type_("button"), ..attributes],
    children,
  )
}

/// Primary action button.
pub fn primary(
  attributes: List(Attribute(msg)),
  children: List(Element(msg)),
) -> Element(msg) {
  button(attributes, Primary, children)
}

/// Secondary action button.
pub fn secondary(
  attributes: List(Attribute(msg)),
  children: List(Element(msg)),
) -> Element(msg) {
  button(attributes, Secondary, children)
}

/// Danger/destructive action button.
pub fn danger(
  attributes: List(Attribute(msg)),
  children: List(Element(msg)),
) -> Element(msg) {
  button(attributes, Danger, children)
}

/// Ghost/minimal button.
pub fn ghost(
  attributes: List(Attribute(msg)),
  children: List(Element(msg)),
) -> Element(msg) {
  button(attributes, Ghost, children)
}

// ATTRIBUTES ------------------------------------------------------------------

/// Make button disabled.
pub fn disabled(value: Bool) -> Attribute(msg) {
  attribute.disabled(value)
}

/// Set button type to submit.
pub fn submit() -> Attribute(msg) {
  attribute.type_("submit")
}
```

### Complex Component (Stateful Web Component)

For interactive components with internal state, use Web Components pattern:

**Create `src/components/ui/accordion.gleam` (public API):**

```gleam
//// Accordion component with collapsible sections.
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
////
//// ## Accessibility
////
//// Follows WAI-ARIA accordion pattern with:
//// - Keyboard navigation (Arrow keys, Home, End)
//// - Proper ARIA attributes
//// - Focus management

import gleam/bool
import gleam/list
import gleam/result
import lustre
import lustre/attribute.{type Attribute}
import lustre/element.{type Element}
import lustre/element/keyed
import components/ui/accordion/item
import components/ui/accordion/root

// TYPES -----------------------------------------------------------------------

/// An accordion item with heading and panel.
pub opaque type Item(msg) {
  Item(
    name: String,
    attributes: List(Attribute(msg)),
    heading: Element(msg),
    panel: Element(msg),
  )
}

// REGISTRATION ----------------------------------------------------------------

/// Register all accordion components. Call before using.
pub fn register() -> Result(Nil, lustre.Error) {
  use _ <- result.try(root.register())
  use _ <- result.try(item.register())
  Ok(Nil)
}

// ELEMENTS --------------------------------------------------------------------

/// The accordion container.
///
/// #### Attributes
///
/// [`multiple`](#multiple), [`loop`](#loop), [`default_value`](#default_value).
///
/// #### Events
///
/// [`on_value_change`](#on_value_change)
///
pub fn view(
  attributes: List(Attribute(msg)),
  children: List(Item(msg)),
) -> Element(msg) {
  keyed.element(root.tag, attributes, {
    use Item(name:, attributes:, heading:, panel:) <- list.filter_map(children)
    use <- bool.guard(name == "", Error(Nil))
    
    let html = item.element([item.name(name), ..attributes], [heading, panel])
    Ok(#(name, html))
  })
}

/// Create an accordion item.
pub fn item(
  name name: String,
  attributes attributes: List(Attribute(msg)),
  heading heading: Element(msg),
  panel panel: Element(msg),
) -> Item(msg) {
  Item(name:, attributes:, heading:, panel:)
}

/// Create accordion heading (contains trigger).
pub fn heading(
  attributes: List(Attribute(msg)),
  children: List(Element(msg)),
) -> Element(msg) {
  element.element("accordion-heading", attributes, children)
}

/// Create accordion trigger button.
pub fn trigger(
  attributes: List(Attribute(msg)),
  children: List(Element(msg)),
) -> Element(msg) {
  element.element("accordion-trigger", attributes, children)
}

/// Create accordion panel (collapsible content).
pub fn panel(
  attributes: List(Attribute(msg)),
  children: List(Element(msg)),
) -> Element(msg) {
  element.element("accordion-panel", attributes, children)
}

// ATTRIBUTES ------------------------------------------------------------------

/// Allow multiple items open at once.
pub fn multiple() -> Attribute(msg) {
  attribute.attribute("type", "multiple")
}

/// Only one item open at a time (default).
pub fn single() -> Attribute(msg) {
  attribute.attribute("type", "single")
}

/// Loop keyboard navigation.
pub fn loop(value: Bool) -> Attribute(msg) {
  case value {
    True -> attribute.attribute("loop", "")
    False -> attribute.none()
  }
}

/// Set default open items.
pub fn default_value(items: List(String)) -> Attribute(msg) {
  attribute.attribute("value", string.join(items, " "))
}

// EVENTS ----------------------------------------------------------------------

/// Fired when open items change.
pub fn on_value_change(handler: fn(List(String)) -> msg) -> Attribute(msg) {
  event.on("accordion:change", {
    use items <- decode.field("detail", decode.list(decode.string))
    decode.success(handler(items))
  })
}
```

**Create `src/components/ui/accordion/root.gleam` (internal):**

```gleam
//// Internal: Accordion root web component.
@internal

import gleam/dynamic/decode
import gleam/json
import gleam/set.{type Set}
import lustre
import lustre/attribute.{type Attribute}
import lustre/component
import lustre/effect.{type Effect}
import lustre/element.{type Element}
import lustre/event

// COMPONENT -------------------------------------------------------------------

pub const tag: String = "ui-accordion"

pub fn register() -> Result(Nil, lustre.Error) {
  let comp = lustre.component(init:, update:, view:, options: [
    component.adopt_styles(False),
    
    component.on_attribute_change("type", fn(value) {
      case value {
        "multiple" -> Ok(ParentSetMultiple(True))
        _ -> Ok(ParentSetMultiple(False))
      }
    }),
    
    component.on_attribute_change("loop", fn(_) {
      Ok(ParentToggledLoop)
    }),
  ])
  
  lustre.register(comp, tag)
}

pub fn element(
  attributes: List(Attribute(msg)),
  children: List(Element(msg)),
) -> Element(msg) {
  element.element(tag, attributes, children)
}

// MODEL -----------------------------------------------------------------------

type Model {
  Model(
    open: Set(String),
    multiple: Bool,
    loop: Bool,
  )
}

fn init(_) -> #(Model, Effect(Msg)) {
  #(Model(open: set.new(), multiple: False, loop: False), effect.none())
}

// UPDATE ----------------------------------------------------------------------

type Msg {
  ParentSetMultiple(value: Bool)
  ParentToggledLoop
  ChildToggledItem(name: String, open: Bool)
}

fn update(model: Model, msg: Msg) -> #(Model, Effect(Msg)) {
  case msg {
    ParentSetMultiple(value) -> 
      #(Model(..model, multiple: value), effect.none())
    
    ParentToggledLoop -> 
      #(Model(..model, loop: !model.loop), effect.none())
    
    ChildToggledItem(name, open) -> {
      let next = case open, model.multiple {
        True, True -> set.insert(model.open, name)
        False, True -> set.delete(model.open, name)
        True, False -> set.from_list([name])
        False, False -> set.new()
      }
      
      let effect = event.emit("accordion:change", {
        next |> set.to_list |> json.array(json.string)
      })
      
      #(Model(..model, open: next), effect)
    }
  }
}

// VIEW ------------------------------------------------------------------------

fn view(model: Model) -> Element(Msg) {
  component.default_slot([
    attribute.role("region"),
    event.on("accordion/item:toggle", {
      use name <- decode.subfield(["detail", "name"], decode.string)
      use open <- decode.subfield(["detail", "open"], decode.bool)
      decode.success(ChildToggledItem(name, open))
    }),
  ], [])
}
```

### File Structure

```
src/components/
├── ui.gleam                    # Re-exports, registers all
└── ui/
    ├── button.gleam            # Simple (functions only)
    ├── input.gleam             # Simple (functions only)
    ├── badge.gleam             # Simple (functions only)
    ├── accordion.gleam         # Complex (public API)
    ├── accordion/
    │   ├── root.gleam          # Web component (internal)
    │   ├── item.gleam          # Web component (internal)
    │   └── panel.gleam         # Web component (internal)
    ├── modal.gleam             # Complex (public API)
    └── modal/
        └── ...
```

### Registration Module

**Create `src/components/ui.gleam`:**

```gleam
//// UI component library.
////
//// Call `register()` before using stateful components.

import gleam/result
import lustre
import components/ui/accordion

// Re-export simple components
pub const button = button
pub const input = input
pub const badge = badge

// Re-export complex components
pub const accordion = accordion

/// Register all stateful components.
pub fn register() -> Result(Nil, lustre.Error) {
  use _ <- result.try(accordion.register())
  // Add other stateful components here
  Ok(Nil)
}
```

## Best Practices

1. **Message naming**: Use Subject-Verb-Object (`UserClickedSubmit`, not `Submit`)
2. **Opaque types**: Hide internal structure of complex components
3. **Keyed lists**: Always use `keyed.element` for dynamic lists
4. **Accessibility**: Add ARIA attributes and keyboard navigation
5. **Documentation**: Document attributes, events, and accessibility notes
6. **Controlled props**: Support both controlled and uncontrolled modes

## References

- [Lustre Documentation](https://hexdocs.pm/lustre/)
- [Lustre UI Patterns](https://github.com/lustre-labs/ui)
- [WAI-ARIA Patterns](https://www.w3.org/WAI/ARIA/apg/patterns/)
