---
name: larajax/responses
description: >
  Build AJAX responses with the ajax() helper: targeted DOM patches via
  update(), partials, redirects, reload, flash messages, browserEvent dispatch,
  invalidFields for validation errors, js/css/img asset loading, error/fatal
  status, and exception handling. Load this skill whenever a Larajax handler
  needs to return something other than a plain string — patching the DOM,
  flashing a message, redirecting after a save, or surfacing inline form errors.
type: core
library: larajax
library_version: "2.2.2"
requires:
  - larajax/core
sources:
  - "larajax/larajax:src/Classes/AjaxResponse.php"
  - "larajax/larajax:src/Classes/AjaxResponse/HasAccessors.php"
  - "larajax/larajax:src/Classes/AjaxResponse/HasOverrides.php"
  - "larajax/larajax:src/init.php"
---

This skill builds on `larajax/core`. Read it first for handler dispatch and
how returned values are wrapped.

## Setup

The `ajax()` global helper returns a fresh `AjaxResponse` from the container.
Every call is chainable.

```php
public function onSave()
{
    request()->validate(['first_name' => 'required']);

    return ajax()
        ->update(['#message' => 'Save complete!'])
        ->browserEvent('profile:saved', ['id' => 42]);
}
```

`AjaxResponse::wrap()` is what `runAjaxAction` calls under the hood. It
auto-promotes return values:

- An `AjaxResponse` is passed through.
- A `RedirectResponse` becomes a redirect op.
- A `Renderable` / `Arrayable` / `JsonSerializable` is unwrapped into `data`.
- A scalar (`string`, `int`, `bool`, `null`) lands in `data.result`.
- An associative array with selector-shaped keys (`#x`, `.y`, `@z`) is
  converted to DOM ops.

## Core Patterns

### Patch DOM regions with `update()`

`update()` queues `patchDom` ops. The key is the CSS selector, the value is
either the content (string / Renderable) or a full `['target','content','swap']`
spec. Swap modes: `update` (default, innerHTML), `replace` (outerHTML),
`append`, `prepend`, `after`, `before`.

```php
return ajax()->update([
    '#message' => view('partials.flash', ['text' => 'Saved']),
    '#todo-list' => [
        'content' => view('partials.todo-item', ['todo' => $todo]),
        'swap'    => 'append',
    ],
]);
```

### Selector shorthand from a flat array

When the handler returns a plain associative array, keys starting with `#`,
`.`, `@`, `^`, `!`, `=` are reinterpreted as DOM updates. `@` = append,
`^` = prepend, `!` = replace, `=` / `#` / `.` = update.

```php
public function onSave()
{
    return [
        '#status' => 'Saved',
        '@#log'   => view('partials.log-entry'),
    ];
}
```

### Return named partials for `data-request-update`

When the client requests partials via `data-request-update`, the controller
returns them by name. The client then patches them into whatever selectors
the HTML attribute mapped them to.

```php
public function onLoad()
{
    return ajax()
        ->partial('profile-card', view('partials.profile-card'))
        ->partial('activity-row', view('partials.activity-row'));
}
```

### Redirect, reload, flash, dispatch events

```php
return ajax()->redirect('/dashboard');

return ajax()->reload();

return ajax()->flash('success', 'Profile saved.');

return ajax()->browserEvent('profile:saved', ['id' => $profile->id]);
```

`flash()` levels are free-form strings (`info`, `success`, `warning`, `error`)
that the frontend's `FlashMessage` handler maps to UI. `browserEventAsync()`
fires after the response is fully applied; `browserEvent()` fires synchronously
during patching.

### Surface validation errors

`request()->validate()` is enough — `AjaxResponse::exception()` catches
`ValidationException` in `LarajaxController::callAction` and converts it to a
422 response with `invalidFields()`. Skip the manual error pass.

```php
public function onSave()
{
    request()->validate([
        'email'    => 'required|email',
        'password' => 'required|min:8',
    ]);

    return ajax()->flash('success', 'Saved');
}
```

For non-Validator errors, set them explicitly:

```php
return ajax()
    ->error('Could not save', 422)
    ->invalidFields([
        'email' => 'Already in use',
    ]);
```

### Load JS / CSS / images alongside the response

`js()`, `css()`, `img()` queue `loadAssets` ops; the client deduplicates by
URL and injects them before patching. `jsInline()` is for inline scripts
(never deduplicated).

```php
return ajax()
    ->css('/css/chart.css')
    ->js(['/js/chart.js' => ['type' => 'module']])
    ->update(['#chart' => view('partials.chart')]);
```

### Force a raw response (file download, streamed PDF)

`force()` bypasses the AJAX envelope entirely — the client treats it as a
hard response (e.g., browser-initiated download).

```php
return ajax()->force(
    response()->download(storage_path('app/report.pdf'))
);
```

## Common Mistakes

### HIGH: Returning a string and expecting a DOM update

Wrong:

```php
public function onSave()
{
    return '<div class="alert">Saved</div>';
}
```

Correct:

```php
public function onSave()
{
    return ajax()->update([
        '#message' => '<div class="alert">Saved</div>',
    ]);
}
```

`AjaxResponse::wrap()` routes scalar return values into `data.result`, not
into the ops list. The response is `200 OK` with the string sitting under
`data.result` — nothing in the DOM changes. The client only patches when
there are `patchDom` ops.

Source: `src/Classes/AjaxResponse.php:127`

### HIGH: Catching `ValidationException` and rebuilding errors by hand

Wrong:

```php
public function onSave()
{
    $v = Validator::make(request()->all(), [
        'email' => 'required|email',
    ]);

    if ($v->fails()) {
        return ajax()->error('Validation failed');
    }
}
```

Correct:

```php
public function onSave()
{
    request()->validate([
        'email' => 'required|email',
    ]);
}
```

`AjaxResponse::exception()` already maps `ValidationException` to status 422
with field-level errors via `invalidFields()`. Returning a generic
`->error()` loses those errors — the client's inline error rendering has
nothing to bind to.

Source: `src/Classes/AjaxResponse.php:510`

### MEDIUM: Wrong swap mode when appending to a list

Wrong:

```php
return ajax()->update([
    '#todo-list' => view('partials.todo-item', ['todo' => $todo]),
]);
```

Correct:

```php
return ajax()->update([
    '#todo-list' => [
        'content' => view('partials.todo-item', ['todo' => $todo]),
        'swap'    => 'append',
    ],
]);

// or the shorthand:
return ['@#todo-list' => view('partials.todo-item', ['todo' => $todo])];
```

The default swap is `update`, which sets `innerHTML` and wipes existing
children. Adding an item to a list silently replaces the whole list.

Source: `src/Classes/AjaxResponse.php:140`

### MEDIUM: Mixed array — some keys are data, some are selectors

Wrong:

```php
public function onCheck()
{
    return [
        '#status' => 'OK',
        'count'   => 5,
    ];
}
```

Correct:

```php
public function onCheck()
{
    return ajax()
        ->data(['count' => 5])
        ->update(['#status' => 'OK']);
}
```

`wrap()` calls `dataWithUpdateSelectors()` for any associative array — keys
that start with `#`, `.`, `@`, `^`, `!`, `=` become DOM ops, the rest become
`data`. Mixed arrays work but are surprising; an agent later reading the code
sees one return shape doing two unrelated things.

Source: `src/Classes/AjaxResponse.php:562`

## See Also

- `larajax/triggers` — `data-request-update` for partial routing
- `larajax/core` — handler dispatch and `wrap()` behavior
