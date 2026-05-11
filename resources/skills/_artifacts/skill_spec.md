# Larajax — Skill Spec

Larajax is a small AJAX framework for Laravel. It lets developers define AJAX handlers directly in Laravel controllers (as `onXxx` methods) and trigger them from HTML using `data-request` attributes, returning targeted DOM updates instead of JSON-only payloads.

## Domains

| Domain | Description | Skills |
| ------ | ----------- | ------ |
| core | Defining server-side AJAX | larajax/core |
| triggers | Triggering AJAX from HTML | larajax/triggers |
| responses | Shaping AJAX responses | larajax/responses |
| components | Reusable view components | larajax/components |

## Skill Inventory

| Skill | Type | Domain | What it covers | Failure modes |
| ----- | ---- | ------ | -------------- | ------------- |
| larajax/core | core | core | Install, LarajaxController, onXxx handlers, request dispatch | 3 |
| larajax/triggers | core | triggers | data-request attribute family, triggers, polling, file uploads | 3 |
| larajax/responses | core | responses | ajax() builder, DOM updates, partials, redirects, flash, errors, assets | 4 |
| larajax/components | core | components | ViewComponent trait, ComponentContainer, alias::onXxx dispatch | 3 |

## Failure Mode Inventory

### larajax/core (3 failure modes)

| # | Mistake | Priority | Source | Cross-skill? |
| - | ------- | -------- | ------ | ------------ |
| 1 | Handler not named with on prefix and PascalCase | CRITICAL | src/Traits/AjaxController.php:98 | — |
| 2 | Controller does not extend LarajaxController | CRITICAL | src/LarajaxController.php:19 | — |
| 3 | GET request used for an AJAX handler | HIGH | src/Classes/AjaxRequest.php:77 | — |

### larajax/triggers (3 failure modes)

| # | Mistake | Priority | Source | Cross-skill? |
| - | ------- | -------- | ------ | ------------ |
| 1 | data-request-data passed as plain JSON | MEDIUM | resources/src/util/json-parser.js | — |
| 2 | File upload missing data-request-files | HIGH | resources/src/request/options.js:41 | — |
| 3 | data-request-update used with no controller-side partial | HIGH | src/Traits/AjaxController.php:113 | responses |

### larajax/responses (4 failure modes)

| # | Mistake | Priority | Source | Cross-skill? |
| - | ------- | -------- | ------ | ------------ |
| 1 | Returning a string and expecting a DOM update | HIGH | src/Classes/AjaxResponse.php:127 | — |
| 2 | Catching ValidationException and rebuilding errors manually | HIGH | src/Classes/AjaxResponse.php:510 | — |
| 3 | Wrong swap mode for list insertion | MEDIUM | src/Classes/AjaxResponse.php:140 | — |
| 4 | Mixed array (data + selectors) routed as DOM ops | MEDIUM | src/Classes/AjaxResponse.php:562 | — |

### larajax/components (3 failure modes)

| # | Mistake | Priority | Source | Cross-skill? |
| - | ------- | -------- | ------ | ------------ |
| 1 | Component handler called without alias prefix | HIGH | src/Classes/AjaxRequest.php:91 | — |
| 2 | Component class missing ViewComponent trait | CRITICAL | src/Traits/ViewComponent.php | — |
| 3 | Component not registered in $components array | HIGH | src/Classes/ComponentContainer.php:42 | — |

## Tensions

| Tension | Skills | Agent implication |
| ------- | ------ | ----------------- |
| DOM updates vs full-page rendering | responses ↔ triggers | data-request-update only works when a matching partial is returned. |
| Controller-owned vs component-owned handlers | core ↔ components | Picking the wrong style means handlers never dispatch. |

## Cross-References

| From | To | Reason |
| ---- | -- | ------ |
| core | responses | Handlers return AjaxResponse objects. |
| triggers | responses | data-request-update is paired with ajax()->partial(). |
| components | core | Components plug into the same dispatch pipeline. |

## Recommended Skill File Structure

- **Core skills:** larajax/core (foundational), larajax/triggers, larajax/responses, larajax/components

## Composition Opportunities

| Library | Integration points | Composition skill needed? |
| ------- | ------------------ | ------------------------- |
| Laravel | Required peer — controller, validation, routing | No (folded into core) |
| Turbo (resources/src/turbo) | Optional view transitions / page renderer | Future — not in v2.2.2 baseline |
