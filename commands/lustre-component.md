# /gleam-lustre-component - Create Lustre Component

Create a reusable Lustre UI component with consistent styling.

## Usage

```
/gleam-lustre-component <component_name> <variant>
```

## Examples

```
/gleam-lustre-component button primary
/gleam-lustre-component input text
/gleam-lustre-component card default
```

## Workflow

### 1. Create Component Module

Create file in `src/app/components/<component_name>.gleam`:

```gleam
import lustre/attribute.{class, type_}
import lustre/element.{type Element}
import lustre/element/html
import lustre/event

// Primary variant
pub fn view_<component_name>_primary(
  label: String,
  on_click: Option(msg),
  is_disabled: Bool,
) -> Element(msg) {
  let base_classes = "px-4 py-2 rounded-md font-medium transition-colors"
  let variant_classes = "bg-black text-white hover:bg-gray-800"
  let disabled_classes = "opacity-50 cursor-not-allowed"

  let classes = case is_disabled {
    True -> base_classes <> " " <> variant_classes <> " " <> disabled_classes
    False -> base_classes <> " " <> variant_classes
  }

  let attrs = [
    class(classes),
    type_("button"),
    attribute.disabled(is_disabled),
  ]

  let attrs = case on_click, is_disabled {
    Some(msg), False -> [event.on_click(msg), ..attrs]
    _, _ -> attrs
  }

  html.button(attrs, [element.text(label)])
}

// Secondary variant
pub fn view_<component_name>_secondary(
  label: String,
  on_click: Option(msg),
) -> Element(msg) {
  let classes = "px-4 py-2 rounded-md font-medium border border-gray-300 bg-white hover:bg-gray-50 transition-colors"

  let attrs = case on_click {
    Some(msg) -> [class(classes), event.on_click(msg), type_("button")]
    None -> [class(classes), type_("button")]
  }

  html.button(attrs, [element.text(label)])
}
```

### 2. Component Naming Convention

Follow consistent naming pattern:

```gleam
// Pattern: view_<component>_<variant>
pub fn view_button_primary(...)    // Primary button
pub fn view_button_secondary(...)  // Secondary button
pub fn view_button_danger(...)     // Danger button

pub fn view_input_text(...)        // Text input
pub fn view_input_email(...)       // Email input
pub fn view_input_password(...)    // Password input

pub fn view_card_default(...)      // Default card
pub fn view_card_elevated(...)     // Elevated card
```

### 3. Use CSS Variables

Reference design tokens from CSS:

```gleam
// Instead of hardcoded colors
let classes = "bg-[var(--bg-primary)] text-[var(--text-primary)]"

// Or use Tailwind if configured
let classes = "bg-black text-white"
```

### 4. Make Components Generic

Use type parameter for message type:

```gleam
pub fn view_button_primary(
  label: String,
  on_click: Option(msg),  // Generic msg type
  is_disabled: Bool,
) -> Element(msg) {       // Returns Element(msg)
  // Implementation
}

// Can be used with any message type
view_button_primary("Submit", Some(FormSubmitted), False)  // Element(FormMsg)
view_button_primary("Cancel", Some(UserCanceled), False)   // Element(UserMsg)
```

### 5. Support Common Props

Common component props pattern:

```gleam
// Button component signature
pub fn view_button(
  label: String,           // Required: button text
  on_click: Option(msg),   // Optional: click handler
  is_disabled: Bool,       // Required: disabled state
  variant: Variant,        // Required: visual style
) -> Element(msg)

// Input component signature
pub fn view_input(
  label: String,           // Required: field label
  field_name: String,      // Required: field name/id
  field_type: String,      // Required: input type
  placeholder: String,     // Required: placeholder text
  value: String,           // Required: current value
  errors: List(String),    // Required: validation errors
  on_input: fn(String) -> msg,  // Required: input handler
) -> Element(msg)
```

### 6. Handle Optional Interactions

Pattern for optional event handlers:

```gleam
pub fn view_button(
  label: String,
  on_click: Option(msg),
) -> Element(msg) {
  let base_attrs = [
    class("px-4 py-2 rounded-md"),
    type_("button"),
  ]

  let attrs = case on_click {
    Some(msg) -> [event.on_click(msg), ..base_attrs]
    None -> base_attrs
  }

  html.button(attrs, [element.text(label)])
}

// Usage
view_button("Enabled", Some(ButtonClicked))  // Clickable
view_button("Disabled", None)                // Not clickable
```

### 7. Add Component to Index (Optional)

If using a components index file:

```gleam
// In src/app/components.gleam
pub use button.{view_button_primary, view_button_secondary}
pub use input.{view_input_text, view_input_email}
```

### 8. Document Component API

Add module documentation:

```gleam
//// Button components with various styles and states.
////
//// ## Examples
////
//// ```gleam
//// import app/components/button
////
//// button.view_button_primary("Submit", Some(FormSubmitted), False)
//// button.view_button_secondary("Cancel", Some(Canceled), False)
//// ```

import lustre/element.{type Element}

/// Primary action button with black background
pub fn view_button_primary(
  label: String,
  on_click: Option(msg),
  is_disabled: Bool,
) -> Element(msg) {
  // Implementation
}
```

## Common Component Patterns

### Button Variants

```gleam
pub type ButtonVariant {
  Primary    // Black background, white text
  Secondary  // White background, border
  Danger     // Red background
  Ghost      // Transparent, text only
}

pub fn view_button(
  label: String,
  on_click: Option(msg),
  variant: ButtonVariant,
  is_disabled: Bool,
) -> Element(msg) {
  let variant_classes = case variant {
    Primary -> "bg-black text-white hover:bg-gray-800"
    Secondary -> "bg-white border border-gray-300 hover:bg-gray-50"
    Danger -> "bg-red-600 text-white hover:bg-red-700"
    Ghost -> "bg-transparent text-gray-700 hover:bg-gray-100"
  }

  let base_classes = "px-4 py-2 rounded-md font-medium transition-colors"
  // ... rest of implementation
}
```

### Input with Validation

```gleam
pub fn view_input(
  label: String,
  field_name: String,
  value: String,
  errors: List(String),
  on_input: fn(String) -> msg,
) -> Element(msg) {
  let has_errors = !list.is_empty(errors)

  let input_classes = case has_errors {
    True -> "w-full px-3 py-2 border-2 border-red-300 rounded-md focus:ring-red-500"
    False -> "w-full px-3 py-2 border border-gray-300 rounded-md focus:ring-black"
  }

  html.div([class("mb-4")], [
    html.label([class("block text-sm font-medium mb-1")], [
      element.text(label),
    ]),
    html.input([
      attribute.name(field_name),
      attribute.value(value),
      class(input_classes),
      event.on_input(on_input),
    ]),
    ..case has_errors {
      True ->
        list.map(errors, fn(error) {
          html.p([class("text-sm text-red-600 mt-1")], [element.text(error)])
        })
      False -> []
    },
  ])
}
```

### Status Badge

```gleam
pub type Status {
  Active
  Pending
  Error
}

pub fn view_badge(status: Status) -> Element(msg) {
  let #(label, bg, text) = case status {
    Active -> #("Active", "bg-green-100", "text-green-800")
    Pending -> #("Pending", "bg-yellow-100", "text-yellow-800")
    Error -> #("Error", "bg-red-100", "text-red-800")
  }

  html.span(
    [class("px-2 py-1 rounded-full text-xs font-medium " <> bg <> " " <> text)],
    [element.text(label)],
  )
}
```

### Loading Spinner

```gleam
pub fn view_loading_spinner(text: String) -> Element(msg) {
  html.div([class("flex flex-col items-center justify-center p-8")], [
    html.div(
      [class("animate-spin rounded-full h-12 w-12 border-b-2 border-gray-900 mb-4")],
      [],
    ),
    html.p([class("text-gray-600")], [element.text(text)]),
  ])
}
```

### Modal

```gleam
pub fn view_modal(
  is_visible: Bool,
  title: String,
  content: Element(msg),
  on_close: msg,
) -> Element(msg) {
  case is_visible {
    False -> element.none()

    True ->
      html.div(
        [
          class("fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50"),
          event.on_click(on_close),
        ],
        [
          html.div(
            [
              class("bg-white rounded-lg p-6 max-w-md w-full"),
              event.on_click(fn(e) { event.stop_propagation(e) }),
            ],
            [
              html.div([class("flex justify-between items-center mb-4")], [
                html.h2([class("text-xl font-semibold")], [element.text(title)]),
                html.button(
                  [class("text-gray-500 hover:text-gray-700"), event.on_click(on_close)],
                  [element.text("✕")],
                ),
              ]),
              content,
            ],
          ),
        ],
      )
  }
}
```

### Empty State

```gleam
pub fn view_empty_state(
  icon: Element(msg),
  title: String,
  description: String,
  action: Option(#(String, msg)),
) -> Element(msg) {
  html.div([class("flex flex-col items-center justify-center p-12 text-center")], [
    html.div([class("mb-4 text-gray-400")], [icon]),
    html.h3([class("text-lg font-semibold text-gray-900 mb-2")], [
      element.text(title),
    ]),
    html.p([class("text-gray-600 mb-6")], [element.text(description)]),
    ..case action {
      Some(#(label, on_click)) -> [
        view_button_primary(label, Some(on_click), False),
      ]
      None -> []
    },
  ])
}
```

## Best Practices

1. **Consistency**: Use same naming pattern for all components
2. **Reusability**: Make components generic with type parameters
3. **Accessibility**: Use semantic HTML and ARIA attributes
4. **Design System**: Reference CSS variables, not hardcoded values
5. **Documentation**: Document component API and provide examples
6. **Type Safety**: Leverage Gleam's type system for props
7. **Variants**: Use custom types for variant options
8. **Error States**: Support error/disabled/loading states

## References

- [Lustre Documentation](https://hexdocs.pm/lustre/)
- [Lustre Patterns](../patterns/lustre-patterns.md)
- [Design System Guidelines](../contexts/lustre-dev.md#design-system-guidelines)
