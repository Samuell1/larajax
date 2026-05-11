---
name: larajax/triggers
description: >
  Wire HTML elements to Larajax AJAX handlers using the data-request attribute
  family. Load this skill when adding data-request, data-request-data,
  data-request-update, data-request-validate, data-request-files,
  data-request-bulk, data-request-trigger (input changed delay), data-request-poll,
  data-request-confirm, data-request-redirect, or any data-request-* attribute to
  Blade/HTML, when debouncing form inputs, when polling, or when uploading files
  through Larajax.
type: core
library: larajax
library_version: "2.2.2"
requires:
  - larajax/core
sources:
  - "larajax/larajax:resources/src/core/request-builder.js"
  - "larajax/larajax:resources/src/core/trigger.js"
  - "larajax/larajax:resources/src/request/options.js"
  - "larajax/larajax:resources/src/request/data.js"
---

This skill builds on `larajax/core`. Read it first for handler naming and the
server-side dispatch model.

## Setup

`data-request` is the only required attribute — its value is the handler name
to invoke. Every other `data-request-*` attribute is optional and modifies the
request.

```html
<button data-request="onSave">Save</button>

<form data-request="onSubmit">
    <input name="first_name">
    <button type="submit">Submit</button>
</form>
```

The default trigger event is derived from the element: `form` → `submit`,
`a` / `button` → `click`, `select` → `change`, `input[type=checkbox|radio|file]`
→ `change`. Override with `data-request-trigger`.

## Core Patterns

### Pass extra data with `data-request-data`

The attribute accepts a relaxed JSON-ish syntax (unquoted keys, single quotes
allowed). Data merges with the form's serialized fields. Parent
`data-request-data` attributes are walked outermost-first, with the element's
own attribute taking priority.

```html
<div data-request-data="context: 'profile'">
    <button data-request="onSave" data-request-data="id: 42">Save</button>
</div>
```

### Patch DOM regions on response with `data-request-update`

The value is an object mapping partial name → target selector + swap modifier.
The leading character on the selector controls the swap mode: `#`/`.`/none =
update, `@` = append, `^` = prepend, `!` = replace, `=` = update (explicit).

```html
<button
    data-request="onLoad"
    data-request-update="profile-card: '#profile', activity-row: '@#activity'">
    Load
</button>
```

The controller has to produce matching partials (see `larajax/responses`):

```php
public function onLoad()
{
    return ajax()
        ->partial('profile-card', view('partials.profile-card'))
        ->partial('activity-row', view('partials.activity-row'));
}
```

### Debounce input, poll an endpoint, fire once on reveal

`data-request-trigger` takes `event modifier modifier:value` strings.
`data-request-poll` fires on an interval (in ms or `1s` syntax) while the
element is visible.

```html
<!-- Debounce search input by 300ms, only when value changes -->
<input
    name="q"
    data-request="onSearch"
    data-request-trigger="input changed delay:300">

<!-- Poll every 5 seconds -->
<div data-request="onTick" data-request-poll="5s"></div>

<!-- Fire when scrolled into view, once -->
<div data-request="onLazyLoad" data-request-trigger="intersect once"></div>
```

### Confirm, redirect, validate, flash

```html
<button data-request="onDelete" data-request-confirm="Delete this item?">Delete</button>

<form data-request="onLogin" data-request-redirect="/dashboard">
    <!-- ... -->
</form>

<form data-request="onSubmit" data-request-validate>
    <!-- forces client-side validity check before sending -->
</form>

<button data-request="onSave" data-request-flash>Save with flash messages</button>
```

### Upload files

Set `data-request-files` to switch the request body to `FormData` so
`<input type="file">` values are included. Without it, file inputs are
silently dropped from the `application/x-www-form-urlencoded` body.

```html
<form data-request="onUpload" data-request-files>
    <input type="file" name="avatar">
    <button type="submit">Upload</button>
</form>
```

### Send JSON instead of form-encoded data

`data-request-bulk` switches the body to `application/json`. Required when
posting nested data that the default form encoder can't represent.

```html
<button
    data-request="onSync"
    data-request-bulk
    data-request-data="payload: {items: [1,2,3]}">
    Sync
</button>
```

## Common Mistakes

### HIGH: File upload missing `data-request-files`

Wrong:

```html
<form data-request="onUpload">
    <input type="file" name="avatar">
    <button type="submit">Upload</button>
</form>
```

Correct:

```html
<form data-request="onUpload" data-request-files>
    <input type="file" name="avatar">
    <button type="submit">Upload</button>
</form>
```

Without the flag, `Options.buildHeaders()` sets
`Content-Type: application/x-www-form-urlencoded` and the request body is
serialized as a query string. File inputs are dropped silently; the handler
receives an empty `request()->file('avatar')`.

Source: `resources/src/request/options.js:41`

### HIGH: `data-request-update` without a matching `ajax()->partial()`

Wrong:

```html
<button data-request="onLoad" data-request-update="profile-card: '#profile'">Load</button>
```

```php
public function onLoad()
{
    return ajax()->data(['ok' => true]);
}
```

Correct:

```php
public function onLoad()
{
    return ajax()->partial('profile-card', view('partials.profile-card'));
}
```

`data-request-update` only sends a list of partial names via the
`X-AJAX-PARTIALS` header. The server still has to render and attach those
partials to the response — the response is otherwise successful, but no DOM
patch is emitted and the client silently does nothing.

Source: `src/Traits/AjaxController.php:113`

### MEDIUM: Strict JSON inside double-quoted attribute breaks the parser

Wrong:

```html
<button data-request="onSave" data-request-data="{"id": 42}">Save</button>
```

Correct:

```html
<button data-request="onSave" data-request-data="id: 42">Save</button>
<!-- or use single-quoted strings: -->
<button data-request="onSave" data-request-data="name: 'Alice'">Save</button>
```

`data-request-data` runs through Larajax's `JsonParser.paramToObj` which
accepts unquoted keys and single-quoted strings. Inline double quotes inside
a double-quoted HTML attribute break the markup, not the parser — use the
relaxed syntax instead of escaping.

Source: `resources/src/util/json-parser.js`

### MEDIUM: Lifecycle hook attributes have to be valid JS expressions

Wrong:

```html
<button
    data-request="onSave"
    data-request-success="alert(I'm done!)">
    Save
</button>
```

Correct:

```html
<button
    data-request="onSave"
    data-request-success="alert('Done')">
    Save
</button>
```

`data-request-success` / `-error` / `-before-send` / `-complete` are evaluated
as JS function bodies via `new Function('context', 'data', attrVal)`. Syntax
errors throw at request time, not at page load — easy to miss in dev.

Source: `resources/src/core/request-builder.js:106`

## See Also

- `larajax/core` — the controller side of these requests
- `larajax/responses` — how to produce partials targeted by `data-request-update`
