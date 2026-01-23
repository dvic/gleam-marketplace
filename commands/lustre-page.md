# /gleam:lustre-page - Create Lustre Page

Create a new page module following idiomatic Model-Update-View patterns.

## Usage

```
/gleam:lustre-page <page_name>
```

## Examples

```
/gleam:lustre-page home
/gleam:lustre-page product_detail
/gleam:lustre-page settings
```

## Workflow

### 1. Create Page Module

Create `src/app/pages/<page_name>.gleam`:

```gleam
//// Home page module.

import gleam/int
import gleam/option.{type Option, None, Some}
import lustre/attribute.{class}
import lustre/effect.{type Effect}
import lustre/element.{type Element}
import lustre/element/html
import lustre/event
import components/ui/button

// TYPES -----------------------------------------------------------------------

pub type Model {
  Model(
    items: RemoteData(List(Item), String),
    selected_id: Option(String),
  )
}

/// Remote data states for async operations.
pub type RemoteData(data, error) {
  NotAsked
  Loading
  Success(data)
  Failure(error)
}

// MESSAGES --------------------------------------------------------------------
// IMPORTANT: Use Subject-Verb-Object naming!

pub type Msg {
  // User interactions
  UserClickedItem(id: String)
  UserClickedRefresh
  UserTypedInSearch(query: String)
  
  // API responses
  ApiReturnedItems(Result(List(Item), String))
  
  // Child component events (if any)
  // ChildEmittedEvent(...)
}

// INIT ------------------------------------------------------------------------

pub fn init() -> #(Model, Effect(Msg)) {
  let model = Model(
    items: Loading,
    selected_id: None,
  )
  
  // Fetch initial data
  #(model, fetch_items())
}

// UPDATE ----------------------------------------------------------------------

pub fn update(model: Model, msg: Msg) -> #(Model, Effect(Msg)) {
  case msg {
    UserClickedItem(id) -> {
      #(Model(..model, selected_id: Some(id)), effect.none())
    }
    
    UserClickedRefresh -> {
      #(Model(..model, items: Loading), fetch_items())
    }
    
    UserTypedInSearch(query) -> {
      // Debounce search or filter locally
      #(model, effect.none())
    }
    
    ApiReturnedItems(Ok(items)) -> {
      #(Model(..model, items: Success(items)), effect.none())
    }
    
    ApiReturnedItems(Error(err)) -> {
      #(Model(..model, items: Failure(err)), effect.none())
    }
  }
}

// VIEW ------------------------------------------------------------------------

pub fn view(model: Model) -> Element(Msg) {
  html.div([class("container mx-auto p-4")], [
    view_header(),
    view_content(model),
  ])
}

fn view_header() -> Element(Msg) {
  html.header([class("mb-6")], [
    html.h1([class("text-2xl font-bold")], [html.text("Page Title")]),
    button.primary(
      [event.on_click(UserClickedRefresh)],
      [html.text("Refresh")],
    ),
  ])
}

fn view_content(model: Model) -> Element(Msg) {
  case model.items {
    NotAsked -> html.text("")
    Loading -> view_loading()
    Success(items) -> view_items(model, items)
    Failure(error) -> view_error(error)
  }
}

fn view_loading() -> Element(Msg) {
  html.div([class("flex justify-center p-8")], [
    html.div([class("animate-spin h-8 w-8 border-2 border-gray-900 rounded-full border-t-transparent")], []),
  ])
}

fn view_items(model: Model, items: List(Item)) -> Element(Msg) {
  // Use keyed rendering for lists!
  element.keyed(html.ul([class("space-y-2")]), {
    list.map(items, fn(item) {
      let is_selected = model.selected_id == Some(item.id)
      #(item.id, view_item(item, is_selected))
    })
  })
}

fn view_item(item: Item, is_selected: Bool) -> Element(Msg) {
  let selected_class = case is_selected {
    True -> " bg-blue-50 border-blue-500"
    False -> " hover:bg-gray-50"
  }
  
  html.li(
    [
      class("p-4 border rounded cursor-pointer" <> selected_class),
      event.on_click(UserClickedItem(item.id)),
      // Accessibility
      attribute.role("button"),
      attribute.tabindex(0),
      attribute.aria_selected(is_selected),
    ],
    [html.text(item.name)],
  )
}

fn view_error(error: String) -> Element(Msg) {
  html.div(
    [
      class("bg-red-50 border border-red-200 rounded p-4"),
      attribute.role("alert"),
    ],
    [
      html.p([class("text-red-800")], [html.text(error)]),
      button.secondary(
        [event.on_click(UserClickedRefresh), class("mt-2")],
        [html.text("Try again")],
      ),
    ],
  )
}

// EFFECTS ---------------------------------------------------------------------

fn fetch_items() -> Effect(Msg) {
  effect.from(fn(dispatch) {
    // Make API call here
    // Example with rsvp:
    // rsvp.get(url, rsvp.expect_json(items_decoder(), ApiReturnedItems))
    Nil
  })
}
```

### 2. Integrate with Main App

**In `src/app.gleam`:**

```gleam
import app/pages/home

// Add to Model
pub type Model {
  Model(
    route: Route,
    home_page: home.Model,
    // ... other pages
  )
}

// Add to Msg (use consistent naming!)
pub type Msg {
  // Router events
  RouterChangedRoute(Route)
  
  // Page messages (wrap child messages)
  HomePageMsg(home.Msg)
  // ... other pages
}

// Init
pub fn init(_) -> #(Model, Effect(Msg)) {
  let #(home_model, home_effect) = home.init()
  
  #(
    Model(
      route: RouteHome,
      home_page: home_model,
    ),
    home_effect |> effect.map(HomePageMsg),
  )
}

// Update - delegate to page
pub fn update(model: Model, msg: Msg) -> #(Model, Effect(Msg)) {
  case msg {
    RouterChangedRoute(route) -> {
      // Handle route change, maybe trigger page init effects
      #(Model(..model, route: route), effect_for_route(route))
    }
    
    HomePageMsg(page_msg) -> {
      let #(page_model, page_effect) = home.update(model.home_page, page_msg)
      #(
        Model(..model, home_page: page_model),
        page_effect |> effect.map(HomePageMsg),  // Don't forget to map!
      )
    }
  }
}

// View - render current page
pub fn view(model: Model) -> Element(Msg) {
  html.main([], [
    case model.route {
      RouteHome -> 
        home.view(model.home_page) 
        |> element.map(HomePageMsg)  // Don't forget to map!
      
      RouteNotFound -> 
        view_not_found()
    }
  ])
}
```

### 3. Common Page Patterns

#### List Page with Pagination

```gleam
pub type Model {
  Model(
    items: RemoteData(PaginatedList(Item), String),
    page: Int,
    per_page: Int,
    search_query: String,
  )
}

pub type Msg {
  UserTypedInSearch(String)
  UserClickedNextPage
  UserClickedPreviousPage
  UserSelectedPerPage(Int)
  ApiReturnedItems(Result(PaginatedList(Item), String))
}

fn update(model: Model, msg: Msg) -> #(Model, Effect(Msg)) {
  case msg {
    UserClickedNextPage -> {
      let next_page = model.page + 1
      #(
        Model(..model, page: next_page, items: Loading),
        fetch_items(next_page, model.per_page, model.search_query),
      )
    }
    // ...
  }
}
```

#### Detail Page with Edit Mode

```gleam
pub type Model {
  Model(
    item: RemoteData(Item, String),
    edit_form: Option(EditForm),
    is_saving: Bool,
  )
}

pub type Msg {
  UserClickedEdit
  UserClickedCancel
  UserClickedSave
  UserUpdatedField(field: Field, value: String)
  ApiReturnedItem(Result(Item, String))
  ApiSavedItem(Result(Item, String))
}

fn view(model: Model) -> Element(Msg) {
  case model.item, model.edit_form {
    Success(item), None -> view_detail(item)
    Success(_), Some(form) -> view_edit_form(form, model.is_saving)
    Loading, _ -> view_loading()
    Failure(err), _ -> view_error(err)
    NotAsked, _ -> element.none()
  }
}
```

#### Form Page with Validation

```gleam
pub type Model {
  Model(
    form: Form,
    errors: Dict(String, List(String)),
    is_submitting: Bool,
  )
}

pub type Msg {
  UserUpdatedField(field: String, value: String)
  UserBlurredField(field: String)  // Validate on blur
  UserSubmittedForm
  ApiReturnedSuccess(Result(Response, String))
}

fn update(model: Model, msg: Msg) -> #(Model, Effect(Msg)) {
  case msg {
    UserUpdatedField(field, value) -> {
      let form = update_form_field(model.form, field, value)
      // Clear error when user starts typing
      let errors = dict.delete(model.errors, field)
      #(Model(..model, form: form, errors: errors), effect.none())
    }
    
    UserBlurredField(field) -> {
      // Validate single field
      let errors = validate_field(model.form, field, model.errors)
      #(Model(..model, errors: errors), effect.none())
    }
    
    UserSubmittedForm -> {
      // Validate all fields
      let errors = validate_form(model.form)
      case dict.is_empty(errors) {
        True -> #(
          Model(..model, is_submitting: True),
          submit_form(model.form),
        )
        False -> #(Model(..model, errors: errors), effect.none())
      }
    }
    
    // ...
  }
}
```

## Message Naming Cheatsheet

```gleam
type Msg {
  // User interactions - always start with User
  UserClickedButton
  UserTypedInField(value: String)
  UserSelectedOption(id: String)
  UserToggledCheckbox(checked: Bool)
  UserSubmittedForm
  UserPressedEnter
  UserScrolledToBottom
  
  // API responses - always start with Api
  ApiReturnedData(Result(Data, Error))
  ApiCreatedItem(Result(Item, Error))
  ApiDeletedItem(Result(Nil, Error))
  
  // Timer/Browser events
  TimerFired
  WindowResized(width: Int, height: Int)
  UrlChanged(Uri)
  
  // Child component events
  ChildSelectedItem(id: String)
  ModalClosedWithResult(Result)
}
```

## Best Practices

1. **Message naming**: ALWAYS use Subject-Verb-Object (`UserClickedSave`, not `Save`)
2. **Effect mapping**: ALWAYS map effects when delegating (`effect.map(PageMsg)`)
3. **Element mapping**: ALWAYS map elements when rendering child views
4. **RemoteData**: Use for ALL async data to handle loading/error states
5. **Keyed lists**: ALWAYS use `element.keyed` for dynamic lists
6. **Accessibility**: Add ARIA attributes, keyboard support
7. **Error handling**: Handle ALL error states in view

## Common Mistakes

```gleam
// ❌ WRONG: Forgot to map effect
HomePageMsg(page_msg) -> {
  let #(page_model, page_effect) = home.update(...)
  #(Model(..model, home_page: page_model), page_effect)  // BUG!
}

// ✅ CORRECT: Map the effect
HomePageMsg(page_msg) -> {
  let #(page_model, page_effect) = home.update(...)
  #(Model(..model, home_page: page_model), page_effect |> effect.map(HomePageMsg))
}

// ❌ WRONG: Forgot to map element
RouteHome -> home.view(model.home_page)  // BUG!

// ✅ CORRECT: Map the element
RouteHome -> home.view(model.home_page) |> element.map(HomePageMsg)

// ❌ WRONG: Imperative message name
type Msg { Submit }

// ✅ CORRECT: Descriptive message name
type Msg { UserSubmittedForm }
```

## References

- [Lustre Documentation](https://hexdocs.pm/lustre/)
- [Lustre Examples](https://github.com/lustre-labs/lustre/tree/main/examples)
- [Elm Architecture Guide](https://guide.elm-lang.org/architecture/)
