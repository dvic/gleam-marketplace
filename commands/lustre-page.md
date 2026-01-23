# /gleam-lustre-page - Create Lustre Page

Create a new page module following the Model-View-Update pattern.

## Usage

```
/gleam-lustre-page <page_name>
```

## Examples

```
/gleam-lustre-page home
/gleam-lustre-page product_detail
/gleam-lustre-page checkout
```

## Workflow

### 1. Create Page Module

Create file in `src/app/pages/<page_name>.gleam`:

```gleam
import gleam/list
import gleam/option.{type Option}
import lustre/attribute.{class}
import lustre/element.{type Element}
import lustre/element/html
import lustre/event
import app/components/button
import app/components/input

// Page Model - State specific to this page
pub type Model {
  Model(
    // Page-specific state
    items: List(Item),
    selected_id: Option(String),
    is_loading: Bool,
    error: Option(String),
  )
}

// Page Messages - Events that can occur on this page
pub type Msg {
  UserClickedItem(id: String)
  UserSubmittedForm(FormData)
  ApiReturnedData(Result(List(Item), String))
}

// Initialize page state
pub fn init() -> Model {
  Model(
    items: [],
    selected_id: option.None,
    is_loading: False,
    error: option.None,
  )
}

// Update page state
pub fn update(model: Model, msg: Msg) -> #(Model, effect.Effect(Msg)) {
  case msg {
    UserClickedItem(id) -> #(
      Model(..model, selected_id: option.Some(id)),
      effect.none(),
    )

    UserSubmittedForm(data) -> #(
      Model(..model, is_loading: True, error: option.None),
      submit_form_effect(data),
    )

    ApiReturnedData(Ok(items)) -> #(
      Model(..model, items: items, is_loading: False),
      effect.none(),
    )

    ApiReturnedData(Error(err)) -> #(
      Model(..model, error: option.Some(err), is_loading: False),
      effect.none(),
    )
  }
}

// Render page
pub fn view(model: Model) -> Element(Msg) {
  html.div([class("container mx-auto p-4")], [
    view_header(),
    case model.is_loading {
      True -> view_loading()
      False -> view_content(model)
    },
    case model.error {
      option.Some(err) -> view_error(err)
      option.None -> element.none()
    },
  ])
}

// View helpers
fn view_header() -> Element(Msg) {
  html.h1([class("text-2xl font-bold mb-4")], [
    element.text("Page Title"),
  ])
}

fn view_loading() -> Element(Msg) {
  html.div([class("text-center py-8")], [
    element.text("Loading..."),
  ])
}

fn view_content(model: Model) -> Element(Msg) {
  html.div([], [
    // Page content
  ])
}

fn view_error(error: String) -> Element(Msg) {
  html.div([class("bg-red-50 border border-red-200 rounded p-4 mb-4")], [
    html.p([class("text-red-800")], [element.text(error)]),
  ])
}

// Effects
fn submit_form_effect(data: FormData) -> effect.Effect(Msg) {
  effect.from(fn(dispatch) {
    // Make API call
    // dispatch(ApiReturnedData(result))
  })
}
```

### 2. Add Route for Page

In `src/app/router.gleam`:

```gleam
pub type Route {
  // ... existing routes
  Route<PageName>
  Route<PageName>Detail(id: String)  // If page has ID parameter
}

pub fn parse_url(uri: Uri) -> Route {
  case uri.path_segments(uri.path) {
    // ... existing routes
    ["<page_name>"] -> Route<PageName>
    ["<page_name>", id] -> Route<PageName>Detail(id)
    _ -> RouteNotFound
  }
}

pub fn to_path(route: Route) -> String {
  case route {
    // ... existing routes
    Route<PageName> -> "/<page_name>"
    Route<PageName>Detail(id) -> "/<page_name>/" <> id
  }
}
```

### 3. Add Page State to Main Model

In `src/app.gleam`:

```gleam
pub type Model {
  Model(
    // ... existing fields
    <page_name>_page: <page_name>.Model,
  )
}

pub fn init(_) -> #(Model, Effect(Msg)) {
  #(
    Model(
      // ... existing fields
      <page_name>_page: <page_name>.init(),
    ),
    effect.none(),
  )
}
```

### 4. Add Page Messages to Main Msg

In `src/app.gleam`:

```gleam
pub type Msg {
  // ... existing messages
  <PageName>Msg(<page_name>.Msg)
}
```

### 5. Handle Page Messages in Main Update

In `src/app.gleam`:

```gleam
pub fn update(model: Model, msg: Msg) -> #(Model, Effect(Msg)) {
  case msg {
    // ... existing cases

    <PageName>Msg(page_msg) -> {
      let #(updated_page, page_effect) =
        <page_name>.update(model.<page_name>_page, page_msg)

      #(
        Model(..model, <page_name>_page: updated_page),
        page_effect |> effect.map(<PageName>Msg),
      )
    }
  }
}
```

### 6. Add Page to Main View

In `src/app.gleam`:

```gleam
pub fn view(model: Model) -> Element(Msg) {
  case model.route {
    // ... existing cases

    Route<PageName> -> view_<page_name>_page(model)
    Route<PageName>Detail(id) -> view_<page_name>_detail_page(model, id)
  }
}

fn view_<page_name>_page(model: Model) -> Element(Msg) {
  <page_name>.view(model.<page_name>_page)
  |> element.map(<PageName>Msg)
}
```

### 7. Add Route Effect (if needed)

If the page needs to load data on navigation:

```gleam
// In src/app.gleam
fn effect_for_route(route: Route) -> Effect(Msg) {
  case route {
    // ... existing cases

    Route<PageName> -> {
      // Trigger page initialization effect
      <page_name>.fetch_data()
      |> effect.map(<PageName>Msg)
    }

    Route<PageName>Detail(id) -> {
      // Fetch specific item
      <page_name>.fetch_item(id)
      |> effect.map(<PageName>Msg)
    }

    _ -> effect.none()
  }
}
```

## Page Patterns

### List Page Pattern

**Pattern**: Display a list of items with filtering

```gleam
pub type Model {
  Model(
    items: RemoteData(List(Item), String),
    search_query: String,
    selected_filter: Filter,
    page: Int,
    per_page: Int,
  )
}

pub type Msg {
  UserSearched(String)
  UserSelectedFilter(Filter)
  UserClickedNextPage
  UserClickedPrevPage
  ApiReturnedItems(Result(PaginatedItems, String))
}

pub fn view(model: Model) -> Element(Msg) {
  html.div([], [
    view_search_bar(model.search_query),
    view_filters(model.selected_filter),
    case model.items {
      NotAsked -> view_initial_state()
      Loading -> view_loading_spinner()
      Success(items) -> view_items_list(items)
      Failure(err) -> view_error_banner(err)
    },
    view_pagination(model.page, model.per_page),
  ])
}
```

### Detail Page Pattern

**Pattern**: Display single item with actions

```gleam
pub type Model {
  Model(
    item: RemoteData(Item, String),
    is_editing: Bool,
    form_data: Option(FormData),
  )
}

pub type Msg {
  UserClickedEdit
  UserClickedCancel
  UserClickedSave
  UserUpdatedField(field: String, value: String)
  ApiReturnedItem(Result(Item, String))
  ApiUpdatedItem(Result(Item, String))
}

pub fn view(model: Model) -> Element(Msg) {
  case model.item {
    Loading -> view_loading_spinner()

    Success(item) ->
      case model.is_editing {
        True -> view_edit_form(model.form_data)
        False -> view_item_details(item)
      }

    Failure(err) -> view_error_state(err)

    NotAsked -> element.none()
  }
}
```

### Form Page Pattern

**Pattern**: Multi-step form with validation

```gleam
import formal/form

pub type Model {
  Model(
    current_step: Step,
    form: form.Form(FormData),
    errors: Dict(String, List(String)),
    is_submitting: Bool,
  )
}

pub type Step {
  StepPersonalInfo
  StepAddress
  StepPayment
  StepReview
}

pub type Msg {
  UserClickedNext
  UserClickedBack
  UserUpdatedField(field: String, value: String)
  UserSubmittedForm
  ApiReturnedSuccess(Result(Nil, String))
}

pub fn view(model: Model) -> Element(Msg) {
  html.div([class("max-w-2xl mx-auto")], [
    view_step_indicator(model.current_step),
    case model.current_step {
      StepPersonalInfo -> view_personal_info_form(model)
      StepAddress -> view_address_form(model)
      StepPayment -> view_payment_form(model)
      StepReview -> view_review_step(model)
    },
    view_navigation_buttons(model),
  ])
}
```

### Dashboard Page Pattern

**Pattern**: Overview with multiple data sources

```gleam
pub type Model {
  Model(
    stats: RemoteData(Stats, String),
    recent_items: RemoteData(List(Item), String),
    notifications: RemoteData(List(Notification), String),
  )
}

pub type Msg {
  UserRefreshedDashboard
  ApiReturnedStats(Result(Stats, String))
  ApiReturnedRecentItems(Result(List(Item), String))
  ApiReturnedNotifications(Result(List(Notification), String))
}

pub fn init() -> #(Model, Effect(Msg)) {
  #(
    Model(stats: Loading, recent_items: Loading, notifications: Loading),
    effect.batch([
      fetch_stats(),
      fetch_recent_items(),
      fetch_notifications(),
    ]),
  )
}

pub fn view(model: Model) -> Element(Msg) {
  html.div([class("grid grid-cols-1 md:grid-cols-3 gap-4")], [
    view_stats_card(model.stats),
    view_recent_items_card(model.recent_items),
    view_notifications_card(model.notifications),
  ])
}
```

## Best Practices

1. **Single Responsibility**: Each page handles one main user flow
2. **Pure Functions**: Keep `update` and `view` pure; effects in `Effect(Msg)`
3. **RemoteData**: Use RemoteData for all async operations
4. **View Helpers**: Break down complex views into helper functions
5. **Effect Mapping**: Always map effects when delegating from parent
6. **Type Safety**: Define clear Model and Msg types
7. **Error Handling**: Handle all error states in view
8. **Loading States**: Show loading indicators during async operations

## Common Gotchas

1. **Effect Mapping**: Don't forget `effect.map(PageMsg)` when delegating
2. **Route Effects**: Remember to trigger effects on route change
3. **State Initialization**: Initialize page state in main app `init()`
4. **Message Exhaustiveness**: Handle all message variants
5. **Element Mapping**: Use `element.map(PageMsg)` when rendering page view

## References

- [Lustre Documentation](https://hexdocs.pm/lustre/)
- [Lustre Patterns](../patterns/lustre-patterns.md)
- [Routing with Modem](../patterns/lustre-patterns.md#routing)
