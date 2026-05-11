---
name: larajax/components
description: >
  Encapsulate reusable AJAX behavior in Larajax view components — classes that
  own onXxx handlers, register on controllers via a $components array (or
  globally via ComponentContainer::$globalComponents), and dispatch via the
  alias::onHandler form of X-AJAX-HANDLER / data-request. Load this skill when
  extracting a widget that owns its own AJAX, when wiring ViewComponent /
  ViewComponentInterface, when sharing handlers across multiple controllers,
  or when a data-request value contains "::".
type: core
library: larajax
library_version: "2.2.2"
requires:
  - larajax/core
sources:
  - "larajax/larajax:src/Traits/ViewComponent.php"
  - "larajax/larajax:src/Contracts/ViewComponentInterface.php"
  - "larajax/larajax:src/Classes/ComponentContainer.php"
  - "larajax/larajax:src/Classes/AjaxRequest.php"
---

This skill builds on `larajax/core`. Read it first for handler naming and
dispatch.

## Setup

A Larajax component is a class that:

1. Implements `ViewComponentInterface`
2. Uses the `ViewComponent` trait (gives it `createIn` and `bindToController`)
3. Is registered on a controller via a `public $components = [...]` array

```php
namespace App\Components;

use Larajax\Contracts\ViewComponentInterface;
use Larajax\Traits\ViewComponent;

class ProfileCard implements ViewComponentInterface
{
    use ViewComponent;

    public function boot()
    {
        // Runs once per request, after registration.
    }

    public function onRefresh()
    {
        return ajax()->partial('profile-card', view('components.profile-card', [
            'user' => auth()->user(),
        ]));
    }
}
```

Register on a controller:

```php
use Larajax\LarajaxController;
use App\Components\ProfileCard;

class ProfileController extends LarajaxController
{
    public $components = [
        ProfileCard::class,
    ];

    public function index()
    {
        return view('pages.profile');
    }
}
```

Trigger from HTML using the `alias::onHandler` form. The default alias is the
short class name (`ProfileCard`); override by passing `['alias' => '...']` to
`createIn`.

```html
<button data-request="ProfileCard::onRefresh">Refresh profile</button>
```

## Core Patterns

### Register a component globally

Components in `ComponentContainer::$globalComponents` register on every
controller automatically. Useful for site-wide widgets (flash queue,
search box, etc.).

```php
// In a service provider's boot()
use Larajax\Classes\ComponentContainer;
use App\Components\GlobalSearch;

ComponentContainer::$globalComponents[] = GlobalSearch::class;
```

### Access component instances from the controller

```php
public function index()
{
    $card = $this->getComponentInstance('ProfileCard');
    // ...
}
```

`__get` on `ComponentContainer` is a shortcut:

```php
$this->componentContainer->ProfileCard;
```

### Compose components from other components

A component can declare its own `$components` array. They are registered when
the parent component is bound to the controller.

```php
class ProfileCard implements ViewComponentInterface
{
    use ViewComponent;

    public $components = [
        AvatarUploader::class,
    ];
}
```

### Component handler that updates only its own partial

```php
public function onRefresh()
{
    return ajax()->partial(
        'profile-card',
        view('components.profile-card', ['user' => auth()->user()])
    );
}
```

Pair with HTML:

```html
<div id="profile-card-wrap">
    @include('components.profile-card')
</div>

<button
    data-request="ProfileCard::onRefresh"
    data-request-update="profile-card: '#profile-card-wrap'">
    Refresh
</button>
```

## Common Mistakes

### CRITICAL: Component class missing the `ViewComponent` trait

Wrong:

```php
class ProfileCard implements ViewComponentInterface
{
    public function onRefresh() { /* ... */ }
}
```

Correct:

```php
class ProfileCard implements ViewComponentInterface
{
    use \Larajax\Traits\ViewComponent;

    public function onRefresh() { /* ... */ }
}
```

`createIn` and `bindToController` come from the trait.
`ComponentContainer::register()` calls `ComponentClass::createIn($controller)`
when iterating `$components`. Without the trait, that static call resolves to
nothing and PHP throws a fatal `Call to undefined method` at request time.

Source: `src/Traits/ViewComponent.php`

### HIGH: Calling a component handler without the alias prefix

Wrong:

```html
<button data-request="onRefresh">Refresh</button>
```

Correct:

```html
<button data-request="ProfileCard::onRefresh">Refresh</button>
```

`getAjaxHandlerName` splits on `::` to determine whether to dispatch to a
component. Without the prefix, `runAjaxAction` looks for `onRefresh` on the
controller itself first, falls back to a linear scan of registered components
(first match wins). The handler may run on the wrong target, or throw
`HandlerNotFound` if no controller method matches.

Source: `src/Classes/AjaxRequest.php:91`

### HIGH: Component not in the controller's `$components` array

Wrong:

```php
class ProfileController extends LarajaxController
{
    // No $components declared
}
```

Correct:

```php
class ProfileController extends LarajaxController
{
    public $components = [
        \App\Components\ProfileCard::class,
    ];
}
```

`ComponentContainer::register()` only iterates `$this->controller->components`
and the static `$globalComponents`. A component that is not in either list is
never instantiated. `data-request="ProfileCard::onRefresh"` then resolves to
`ComponentNotFound: Component name [ProfileCard] not found`.

Source: `src/Classes/ComponentContainer.php:42`

## See Also

- `larajax/core` — controller-level handler dispatch
- `larajax/responses` — building responses from inside a component handler
