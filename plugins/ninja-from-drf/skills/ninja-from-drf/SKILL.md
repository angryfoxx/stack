---
name: ninja-from-drf
description: Migrate an existing Django API from Django REST Framework to django-ninja or django-ninja-extra while preserving routes, request/response contracts, auth, permissions, throttling, pagination, filtering, and test coverage. Use when replacing APIView/ViewSet/GenericAPIView and DRF serializers/routers with ninja Schema DTOs, function-based route handlers, or NinjaExtraAPI/APIController wiring.
---

# Ninja from DRF

## Overview

Migrate `Django REST Framework` transport layer to `django-ninja` incrementally.
Use `django-ninja-extra` when it improves DRF-parity, controller organization, permission ergonomics, dependency injection, or filtering/pagination/searching support.
Keep business logic unchanged unless user explicitly asks.

Required docs (always reference first):
- Django 6 docs: https://docs.djangoproject.com/en/6.0/
- Django async docs: https://docs.djangoproject.com/en/6.0/topics/async/
- django-ninja docs: https://django-ninja.dev
- django-ninja response docs: https://django-ninja.dev/guides/response/
- django-ninja v1 notes: https://django-ninja.dev/whatsnew_v1/
- django-ninja-extra docs: https://eadwincode.github.io/django-ninja-extra/
- DRF docs: https://www.django-rest-framework.org
- Local map: [references/drf-to-ninja-map.md](references/drf-to-ninja-map.md)

## Migration Policy

- Default mode: strict transport parity.
- Preserve by default: paths, methods, request/response schemas, status codes, headers, auth semantics, permission semantics, throttling semantics, pagination envelope, filter/query behavior.
- If user approves drift (for example switching to ninja-native validation errors, changing route grouping, or adopting ninja-extra generated CRUD routes), stop forcing legacy behavior and document this as `approved drift`.
- Never mix implicit drift: every intentional mismatch must be explicitly listed.
- Do not preserve DRF transport internals as implementation strategy.
  Keep observable contracts, but realize them with django-ninja / django-ninja-extra + project-native libraries.
- Do not force `django-ninja-extra` everywhere.
  Use it selectively where its features materially improve parity or maintainability.

## Version Compatibility Gate

Before migration, verify actual installed runtime and package versions. Do not rely only on package classifiers or README examples.

Baseline checked on 2026-04-28:
- Django `>=6.0` requires Python `>=3.12` and supports Python `3.12`, `3.13`, and `3.14`.
- django-ninja `>=1.6.2` is the current Django 6-compatible baseline.
- django-ninja-extra `>=0.31.4` is the current controller/model-controller baseline.

Required commands before editing a slice:
- `uv run python -Wd manage.py check` (or `docker exec <web-container> python -Wd manage.py check` in containerized environments)
- repository-native linters/type checks
- repository-native tests for the target API slice
- smoke test URL wiring for `api.urls`

If `django-ninja-extra` is used, smoke test controller registration, route inheritance, API-level auth inheritance, permissions, throttling, pagination, and model-controller routes in this repository before trusting generated behavior.

## Workflow

### 1. Inventory DRF surface

- Read project API entrypoints and URL wiring first.
- Find usages of: `APIView`, `GenericAPIView`, `ViewSet`, `GenericViewSet`, `ModelViewSet`, `@api_view`, DRF routers, DRF serializer classes, parser/renderer classes, auth/permission/throttle classes, DRF exception handler hooks.
- Build per-endpoint migration matrix:
  - path, method, auth, permissions, throttles, parser/renderer behavior,
  - input DTO, output DTO,
  - pagination/filtering/search contract,
  - expected statuses, expected headers,
  - known error paths.

### 2. Inventory project-native batteries (required)

- Before writing adapters, inspect existing project libraries and integrations
  that should power the migrated transport layer.
- Prefer libraries already used by the project through django-ninja or django-ninja-extra mechanisms over custom wrappers.
- Examples:
  - auth/rbac integrations,
  - rate limiting packages,
  - pagination/filtering/search libraries,
  - dependency injection setup (`injector`, `django_injector`, `NINJA_EXTRA['INJECTOR_MODULES']`),
  - existing controller conventions.

### 3. Freeze observable behavior

- Capture behavior from existing tests + serializers + routing.
- Mark blocking mismatches before coding:
  - route/method changes,
  - schema field changes,
  - auth/permission/throttle changes,
  - pagination/filtering/search response shape changes,
  - parser/renderer negotiation changes,
  - status/header changes.

### 4. Decide error strategy (required gate)

Choose one and record it before edits:

- `strict parity`:
  - keep existing error status/payload semantics,
  - adapt ninja or ninja-extra exception handlers as needed.
- `approved ninja-native errors`:
  - use django-ninja-native error mapping (422 validation errors, `HttpError`, etc.),
  - optionally use `ninja_extra.exceptions.APIException` where a DRF-like exception surface helps,
  - use `ninja.Status(status_code, body)` for explicit status returns on django-ninja 1.6+,
  - do not introduce deprecated `(status_code, body)` tuple returns,
  - update tests and docs expectations accordingly,
  - still preserve happy-path behavior and auth/permission/throttle behavior.

### 5. Choose transport style per slice (required gate)

Pick the smallest tool that preserves the contract cleanly:

- prefer plain `django-ninja` when:
  - handlers are simple,
  - class-based grouping adds little value,
  - no controller-level permission or DI benefit exists,
  - pagination/filtering/searching can be expressed cleanly with native ninja features.
- prefer `django-ninja-extra` when:
  - DRF `ViewSet`/`APIView` grouping maps better to class-based controllers,
  - controller-level or route-level permissions improve parity,
  - dependency injection / service-layer composition is already used or clearly beneficial,
  - `RouteContext` helps with request/response/header handling,
  - built-in searching/ordering/pagination helpers materially reduce custom code,
  - `ModelController` can replace repetitive CRUD without route/schema drift.
- for each slice, record the chosen transport style explicitly.

### 6. Migrate by slice (app or endpoint group)

- Migrate one bounded slice at a time.
- Keep module layout unless architecture/lint contracts require change.
- Do not keep class names that encode DRF transport internals.
  Rename transport classes to neutral or ninja-oriented names.
- Prefer thin transport adapters:
  - keep usecases/services/repositories untouched,
  - change only DTOs/handlers/router/controller wiring.

### 7. Translate serializers and request/response contracts

- Replace `Serializer` / `ModelSerializer` transport DTO usage with `ninja.Schema` subclasses.
- For django-ninja v1+ / Pydantic v2, prefer `ModelSchema.Meta` for generated model schemas.
- Avoid new `class Config` examples unless maintaining existing legacy code.
- Use Pydantic `model_config = ConfigDict(...)` on plain `Schema` when schema config is required.
- When project already uses `ModelSchema`, `ninja.orm`, or `ninja-schema`, evaluate them for parity but verify every generated field explicitly.
- Map request sources explicitly:
  - body: Schema model as function parameter (auto-detected) or explicit `Body(...)`,
  - query: typed function parameters with `Query(...)` or plain annotations,
  - path: typed function parameters matching `{param}` in route path,
  - headers: `Header(...)` annotated parameters,
  - cookies: `Cookie(...)` annotated parameters,
  - files: `File(...)` with `UploadedFile` type,
  - form data: `Form(...)` annotated parameters.
- Split input/output Schemas when response contains server-owned fields.
- Preserve field aliases, required/optional behavior, and validation constraints needed for contract parity.
- DRF `SerializerMethodField` should usually map to `resolve_<field>` methods on response schemas or to explicit response assembly in handlers/controllers.
- Use Pydantic validators for validation or normalization, not as the primary replacement for computed response fields.
- DRF `ModelSerializer` auto-generated fields must be verified field-by-field.
- If using ninja-extra `ModelController`, remember it depends on `ninja-schema`; treat generated schemas as convenience, not proof of parity.

### 8. Translate DRF handlers to django-ninja or django-ninja-extra

- Replace function-based `@api_view` endpoints with `@api.get`/`@api.post`/etc. decorated handlers, `@api.api_operation([...])`, or controller methods only when route parity stays intact.
- Replace `APIView`/`GenericAPIView` with either:
  - dedicated handler functions grouped on a `Router`, or
  - `@api_controller` classes with `@route.get` / `@route.post` / `@route.patch` / etc. (or `@http_get` family) when controller structure is beneficial.
- Replace `ViewSet`/`ModelViewSet` actions with either:
  - handler functions on a `Router`, preserving route paths and HTTP methods per action, or
  - controller methods on `django-ninja-extra` preserving route shape, action-specific auth/permissions/throttles, and route names.
- Replace `@action(detail=True/False, ...)` with dedicated route handlers preserving the detail/list URL shape.
- Use `ModelController` only when generated CRUD operations can preserve existing paths, methods, schemas, and per-action behavior; otherwise write explicit handlers/controllers.
- Keep route names and URL shapes stable unless user requested drift.
- For explicit non-default status returns, use `ninja.Status(status_code, body)` instead of deprecated tuple return syntax.
- For body-less responses, declare `response={204: None}` and return `Status(204, None)` or the project-approved empty-response equivalent.
- Port exception behavior intentionally:
  - strict parity mode: preserve old statuses/payloads via `api.add_exception_handler(...)` and controller-aware handlers where needed,
  - approved drift mode: keep ninja-native or ninja-extra-native errors and update tests.

### 9. Replace API wiring

- Replace DRF router registration and URL wiring with one of:
  - `NinjaAPI(...)` + `Router(...)` + `api.add_router(prefix, router)` + Django URL include, or
  - `NinjaExtraAPI(...)` + `api.register_controllers(...)` for controller-driven slices,
  - mixed mode only when route ownership is unambiguous and documented.
- Keep namespace stability (`api:*` names) unless drift approved.
- Add docs/openapi routes only if project already exposes them or user asks.
  (django-ninja and `NinjaExtraAPI` can expose `/docs` and `/openapi.json`; configure URLs to match existing behavior.)

### 10. Port auth, permissions, and throttling

- Keep auth expectations endpoint-by-endpoint.
- Map DRF `authentication_classes` to ninja / ninja-extra `auth=` parameter (global, router, controller, or route level).
- For ninja-extra, use controller-level `permissions=[...]` and route-level overrides where they better preserve DRF permission structure.
- Map DRF `permission_classes` to explicit permission checks, controller permissions, object-level permission helpers, or auth callables as appropriate.
- Map DRF `throttle_classes` to `throttle=` using supported throttle classes.
- Keep throttling behavior and headers (for example `Retry-After`) unless drift approved.
- Preserve method-specific permission behavior and action-level differences via per-route `auth=` / `permissions=` / `throttle=` overrides.
- Prefer existing project-native auth/throttle libraries where possible.
- Avoid DRF compatibility shims as final solution.
  If a temporary shim is unavoidable, mark it explicitly as temporary,
  document replacement plan, and list it in `unresolved gaps`.

### 11. Port filtering, ordering, searching, and pagination

- Preserve query parameter names, default ordering/search semantics, and response envelope.
- Start with existing contract, not framework defaults.
- For plain django-ninja, use `FilterSchema`, typed query params, or project-native helpers.
- For django-ninja-extra, evaluate:
  - `@paginate(...)`,
  - `@ordering(...)`,
  - `@searching(...)`,
  - `FilterSchema` integration,
  - `NinjaPaginationResponseSchema` where envelope compatibility can be preserved.
- Do not adopt ninja-extra defaults blindly:
  - verify envelope keys,
  - verify query parameter names,
  - verify ordering/search field allowlists,
  - verify generated OpenAPI contracts if relied upon by clients.

### 12. Update tests for ninja tooling

- Replace DRF-specific API test assumptions where necessary with ninja/Django-native testing entrypoints.
- Use `ninja.testing.TestClient` or `ninja.testing.TestAsyncClient` for direct handler/controller testing,
  or use Django's standard `Client`/`AsyncClient` for full-stack integration tests.
- When controller DI is used, ensure tests wire the injector/services the same way production does.
- Migrate API test fixtures and clients as first-class migration scope:
  keep fixture intent stable (`api_client`, auth fixtures),
  but rebind them to the migrated ninja stack.
- Keep contract assertions endpoint-by-endpoint:
  status codes, payloads, headers, auth/permission behavior, throttle behavior,
  parser/renderer behavior, pagination/filtering/search semantics.
- Ensure negative-path tests cover declared error responses.
- Remove persistent DRF transport imports from fully migrated transport/test modules unless explicitly approved as temporary drift.

### 13. Validate sync/async boundaries

- Do not assume Django 6 makes the full Django stack async-native.
- Async handlers only provide stack-level benefit under ASGI and when sync-only middleware does not force thread-per-request adaptation.
- In async ninja handlers/controllers, do not return unevaluated sync `QuerySet` objects.
- Use async ORM methods where available and verified by tests.
- Wrap unavoidable sync ORM or Django internals with `sync_to_async(..., thread_sensitive=True)` from a dedicated sync function.
- Keep async authentication classes paired with async endpoint handlers.
- Test async controller/model-controller slices with `ninja.testing.TestAsyncClient` or Django `AsyncClient` when behavior depends on async execution.

### 14. Validate each slice with repository-native CI entrypoints

- Run the same commands CI uses in this repository.
- Prefer official script entrypoints over ad-hoc command sets.
- If docker-only CI/test environment is required, run in docker.

### 15. Finish gate

Do not mark slice done until:
- linters pass,
- tests pass (or failures are explicitly approved drift),
- report is updated.

## Translation Rules

- Business logic unchanged by default.
- Keep transport deterministic and typed.
- Avoid hidden route normalization changes (trailing slash/prefix).
- Preserve auth, permissions, and throttle semantics even when error payload format drifts.
- Preserve pagination envelope and cursor/page semantics unless drift approved.
- Preserve ordering/search parameter names and field behavior unless drift approved.
- Preserve parser/renderer negotiation or explicitly list approved drift.
- Be explicit about framework constraints (for example ninja-native validation produces 422 with Pydantic error detail).
- Use `Status(status_code, body)` for explicit status responses; do not add deprecated tuple status returns.
- Treat Django async as boundary-sensitive: ASGI, middleware, ORM access pattern, auth, and test client all matter.
- Do not use DRF transport classes in migrated slices unless explicitly approved by the user as temporary.
- Do not keep DRF-bound class names in migrated transport code.
- Do not assume ninja-extra generated routes, schemas, or permissions are parity-safe until verified endpoint-by-endpoint.

## Reporting Format (required)

At each meaningful checkpoint and at completion, report in 3 sections:

1. `preserved behavior`
2. `approved drift`
3. `unresolved gaps`

`unresolved gaps` must include concrete files/endpoints and why unresolved.

## Migration Pitfalls (DRF-to-ninja-derived)

- DRF routers often default to trailing slash behavior; django-ninja and controller registration in ninja-extra do not preserve this automatically. Preserve URL shape exactly or document drift.
- `partial=True` update semantics: DRF serializers support `partial=True` natively; in ninja, use a separate Schema with all `Optional` fields or use Pydantic config with optional fields.
- Pagination response envelopes (`count/next/previous/results` and ordering) must stay stable unless drift approved. django-ninja and ninja-extra pagination helpers have their own default wrappers and may not match DRF's envelope out of the box.
- Content negotiation: DRF supports multiple renderers/parsers; django-ninja defaults to JSON-first behavior. If the DRF API serves non-JSON media types, configure parser/renderer support or document as approved drift.
- Action-level permissions and throttles in `ViewSet` classes must be preserved per route, not only per router/controller. Use per-decorator `auth=` / `permissions=` / `throttle=` overrides.
- Header-bearing responses (for example location or throttle headers) require explicit response handling. In ninja-extra controllers, `RouteContext.response.headers` can help, but header behavior still must be verified against existing tests.
- DRF `ModelSerializer` auto-generates fields from model; `ninja.Schema`, `ModelSchema`, and `ninja-schema` helpers require parity review, especially for aliases, nested fields, and writable semantics.
- DRF nested writable serializers (create/update with nested objects) require manual handling in ninja handlers/controllers unless the project already has a safe abstraction.
- DRF `SerializerMethodField` -> prefer response schema `resolve_<field>` methods or explicit handler/controller assembly. Validators are for validation/normalization, not the default computed-field replacement.
- DRF `HyperlinkedModelSerializer` URL fields have no direct ninja equivalent; compute URLs manually or populate them explicitly in the response model.
- DRF filter backends (`DjangoFilterBackend`, `SearchFilter`, `OrderingFilter`) do not map 1:1. ninja-extra offers `searching` and `ordering` helpers inspired by DRF, but query names and behavior still require contract verification.
- `ModelController` defaults and generated route sets can drift from custom DRF actions, route names, lookup conventions, and serializer behavior; use only where generation is genuinely parity-safe or drift is approved.
- Async auth in ninja-extra requires matching async endpoint handlers; do not mix async auth classes into sync handlers.
- Async Django does not automatically make sync ORM access safe. In async handlers/controllers, avoid unevaluated sync QuerySets and wrap unavoidable sync Django operations with `sync_to_async(..., thread_sensitive=True)`.
- django-ninja 1.6+ deprecates `(status_code, body)` tuple returns; use `Status(status_code, body)`.
- For django-ninja v1+ / Pydantic v2, prefer `ModelSchema.Meta`; avoid adding new `class Config` examples unless maintaining legacy code.
- Dependency injection is useful, but introducing DI during transport migration can become architecture drift if the project does not already use it.

## Output Checklist

- Version compatibility gate completed with installed package versions and `uv run python -Wd manage.py check` (or `docker exec <web-container> python -Wd manage.py check`).
- DRF root wiring replaced with `NinjaAPI` / `NinjaExtraAPI` routing for migrated slices.
- DRF API views/viewsets replaced with explicit ninja handlers or ninja-extra controllers.
- DRF transport serializers replaced with `ninja.Schema`-based DTOs (and only parity-reviewed generated schemas where approved).
- Auth/permission/throttle behavior preserved or explicitly drift-approved.
- Pagination/filtering/searching and parser/renderer behavior preserved or explicitly drift-approved.
- Project-native batteries were evaluated first and used where possible.
- No persistent DRF shim or DRF-bound naming remains in migrated transport layer.
- Tests updated only where needed to keep equivalent behavioral coverage for selected strategy.
- Response contracts verified endpoint-by-endpoint (status codes, payloads, headers).
- Explicit status returns use `Status(...)`, not deprecated tuple syntax.
- Async handlers/controllers have verified ORM access patterns and matching async test coverage where relevant.
- Repository-native CI entrypoints executed per slice.
- Final report emitted in required 3-section format.
- After successful migration and green CI for migrated API surfaces, remove `djangorestframework` from project dependencies unless user explicitly approves keeping it.
