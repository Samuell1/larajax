---
name: larajax/core
description: >
  Install Larajax in a Laravel app, expose AJAX handlers as onXxx methods on a
  LarajaxController, and let larajax route AJAX requests automatically. Load
  this skill when wiring composer require larajax/larajax, npm install larajax,
  jax.start(), extending LarajaxController, adding the AjaxController trait,
  the X-AJAX-HANDLER header, or anytime an "onSave / onDelete / onUpdate"
  controller method is mentioned in a Laravel project.
type: core
library: larajax
library_version: "2.2.2"
sources:
  - "larajax/larajax:src/LarajaxController.php"
  - "larajax/larajax:src/Traits/AjaxController.php"
  - "larajax/larajax:src/Classes/AjaxRequest.php"
  - "larajax/larajax:src/init.php"
  - "larajax/larajax:resources/src/index.js"
  - "larajax/larajax:resources/src/request/options.js"
---

## Setup

Install both halves of the framework — the Composer package handles the
server, the npm package handles the browser.

```bash
composer require larajax/larajax
npm install larajax
```

Bootstrap the frontend in your JS entry file:

```js
import { jax } from 'larajax';
window.jax = jax;
jax.start();
```

Make sure the Laravel CSRF token is exposed in the page head — larajax reads
it from `meta[name="csrf-token"]`:

```blade
<meta name="csrf-token" content="{{ csrf_token() }}">
```

Define a controller and a route. A single `Route::any` URL is enough; larajax
multiplexes handlers via the `X-AJAX-HANDLER` request header.

```php
use Larajax\LarajaxController;

class ProfileController extends LarajaxController
{
    public function index()
    {
        return view('pages.profile');
    }

    public function onSave()
    {
        request()->validate(['first_name' => 'required']);

        return ajax()->update([
            '#message' => 'Save complete!',
        ]);
    }
}
```

```php
// routes/web.php
Route::any('/profile', [ProfileController::class, 'index']);
```

## Core Patterns

### Define handlers as `onXxx` methods

Method names must match `^on[A-Z][a-zA-Z]*$`. The controller's normal action
(`index`) still runs for non-AJAX requests; AJAX requests are intercepted in
`callAction` and dispatched to the matching `onXxx` method.

```php
class TodosController extends LarajaxController
{
    public function index() { return view('todos.index'); }

    public function onCreate() { /* ... */ }
    public function onDelete() { /* ... */ }
    public function onReorder() { /* ... */ }
}
```

### Per-action handler scoping

Prefix a handler with the action name to scope it: `index_onSave` only runs
when the request hits the `index` action. Useful when one controller serves
multiple actions that share handler names.

```php
public function onSave() { /* default */ }
public function edit_onSave() { /* only when calling the edit action */ }
```

### Use the AjaxController trait on an existing base class

When extending a custom base controller instead of `LarajaxController`, add
the trait and forward `callAction` yourself.

```php
use Larajax\Contracts\AjaxControllerInterface;
use Larajax\Traits\AjaxController as AjaxControllerTrait;

class MyBaseController extends Controller implements AjaxControllerInterface
{
    use AjaxControllerTrait;

    public function callAction($action, $parameters)
    {
        try {
            if ($result = $this->callAjaxAction($action, $parameters)) {
                return $result;
            }
        } catch (\Exception $ex) {
            return ajax()->exception($ex);
        }

        return parent::callAction($action, $parameters);
    }
}
```

## Common Mistakes

### CRITICAL: Handler not named with `on` prefix and PascalCase

Wrong:

```php
class ProfileController extends LarajaxController
{
    public function save() { /* never reached */ }
}
```

Correct:

```php
class ProfileController extends LarajaxController
{
    public function onSave() { /* ... */ }
}
```

`runAjaxAction` validates handler names with the regex
`^on[A-Z][a-zA-Z]*$` and throws `HandlerNameInvalid` for anything else
(`onsave`, `on_save`, `save`). The check is strict — the first character
after `on` must be uppercase.

Source: `src/Traits/AjaxController.php:98`

### CRITICAL: Controller does not extend LarajaxController or use the trait

Wrong:

```php
use Illuminate\Routing\Controller;

class ProfileController extends Controller
{
    public function onSave() { /* runs as a normal action, returns raw */ }
}
```

Correct:

```php
use Larajax\LarajaxController;

class ProfileController extends LarajaxController
{
    public function onSave() { /* dispatched by callAjaxAction */ }
}
```

`callAjaxAction` is what bridges the AJAX request to your handler. Without it
the method runs only if Laravel's router resolves the action name to `onSave`
directly — which the route definition does not do.

Source: `src/LarajaxController.php:19`

### HIGH: Route restricted to GET so AJAX handlers never dispatch

Wrong:

```php
Route::get('/profile', [ProfileController::class, 'index']);
```

Correct:

```php
Route::any('/profile', [ProfileController::class, 'index']);
// or
Route::match(['get', 'post'], '/profile', [ProfileController::class, 'index']);
```

`AjaxRequest::hasAjaxHandler()` checks both `$request->ajax()` and
`$request->method() === 'POST'`. A GET-only route silently bypasses every
handler — the page loads, the AJAX request returns the index view's HTML,
and nothing in the DOM updates.

Source: `src/Classes/AjaxRequest.php:77`

## See Also

- `larajax/triggers` — how HTML attributes invoke these handlers
- `larajax/responses` — what handlers should return
- `larajax/components` — extracting handlers into reusable components
