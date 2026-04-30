# DRF to django-ninja / django-ninja-extra mapping

Use this mapping when replacing transport-layer code.

## Core constructs

- `APIView` / `GenericAPIView` -> either function-based handlers with `@api.get`, `@api.post`, `@api.put`, `@api.patch`, `@api.delete`, or `@api_controller` methods with `@route.get`, `@route.post`, `@route.put`, `@route.patch`, `@route.delete`
- `ViewSet` / `GenericViewSet` / `ModelViewSet` -> either `Router` with grouped handler functions or `@api_controller` class with grouped controller methods
- `@action(detail=..., methods=[...], url_path=..., url_name=...)` on `ViewSet` -> dedicated route decorator on `Router` or dedicated controller route preserving detail/list scope, HTTP methods, and route name/path contract
- `@api_view([...])` -> `@api.api_operation([...], "/path")`, individual method decorators (`@api.get`, `@api.post`, ...), or controller methods when grouping is beneficial and contract-safe
- DRF router registration (`DefaultRouter`, `SimpleRouter`) -> `NinjaAPI(...)` + `Router(...)` + `api.add_router(prefix, router)` or `NinjaExtraAPI(...)` + `api.register_controllers(...)`
- DRF serializer classes (`Serializer`, `ModelSerializer`) -> `ninja.Schema`; optionally parity-reviewed `ModelSchema` / `ninja-schema` helpers where already used or explicitly approved
- DRF `SerializerMethodField` -> prefer response schema `resolve_<field>` methods or explicit handler/controller response assembly
- Pydantic v2 / django-ninja v1+ model schema config -> prefer `ModelSchema.Meta`; avoid new `class Config` examples unless maintaining legacy code
- DRF exception handler hooks -> `api.add_exception_handler(ExcClass, handler_func)`
- DRF `APIException` ergonomics -> `ninja_extra.exceptions.APIException` when a DRF-like exception surface is useful

## Controller-specific django-ninja-extra features

- class-based transport grouping -> `@api_controller("/prefix", tags=[...], auth=..., permissions=[...])`
- controller registration -> `api.register_controllers(MyController)`
- controller route methods -> `@route.get`, `@route.post`, `@route.put`, `@route.patch`, `@route.delete`, `@route.generic`
- request/response context -> `self.context` / `RouteContext`
- response header mutation -> `self.context.response.headers[...] = ...`
- access current request/user/auth/kwargs -> `self.context.request`, `self.context.request.user`, `self.context.request.auth`, `self.context.kwargs`
- object permission checks -> controller helpers such as `check_object_permission(...)`
- DI-oriented controller/service composition -> constructor/service injection with `injector` / `django_injector` when the project already uses it
- CRUD-heavy resources -> `ModelController` only when generated routes and schemas can preserve existing behavior

## Inputs and outputs

- request body serializer -> `ninja.Schema` subclass as function parameter (auto-detected as body)
- query params (`request.query_params`) -> typed function parameters with `Query(...)` or plain type annotations
- path params (`kwargs`) -> typed function parameters matching `{param}` in path string or available through `self.context.kwargs` in controllers
- headers -> `Header(...)` annotated function parameters
- cookies -> `Cookie(...)` annotated function parameters
- file uploads (`request.FILES`) -> `File(...)` with `UploadedFile` type annotation
- form data -> `Form(...)` annotated function parameters
- serializer output -> `response=SchemaClass` on route decorator
- multiple response schemas/statuses -> `response={status_code: SchemaOrNone}` with `ninja.Status(status_code, body)` returns
- `204 No Content` -> declare `response={204: None}` and keep response body empty
- header-bearing responses -> explicit response object handling; in controllers, `RouteContext.response.headers` can be used

## Auth, permissions, throttling

- authentication classes -> ninja / ninja-extra `auth=` parameter at global (`NinjaAPI(auth=...)`, `NinjaExtraAPI(auth=...)`), router, controller, or route level
- permission classes (class/action specific) -> explicit permission checks or `permissions=[...]` on `@api_controller` with route-level overrides where needed
- object-level permissions -> explicit checks in handlers/controllers; in ninja-extra controllers, use controller permission helpers where appropriate
- throttle classes -> `throttle=` at global, router, controller, or route level using supported throttle classes
- multiple throttles -> supported in django-ninja-extra when multiple rate constraints must be preserved
- `@action` overrides (`permission_classes`, `authentication_classes`, `throttle_classes`, `serializer_class`) -> per-route `auth=`, `permissions=`, `throttle=`, and `response=` / explicit request schema on individual handlers or controller methods

## Pagination, filtering, ordering, searching

- DRF pagination classes -> plain django-ninja pagination or django-ninja-extra `@paginate(...)`; verify response envelope parity
- DRF `DjangoFilterBackend` -> `FilterSchema` or explicit typed query parsing
- DRF `SearchFilter` -> explicit query logic or django-ninja-extra `@searching(...)`; verify query parameter and matching semantics
- DRF `OrderingFilter` -> explicit ordering logic or django-ninja-extra `@ordering(...)`; verify allowed fields and param name
- DRF `ListModelMixin` style list endpoints -> explicit list handler/controller method, or `ModelController` list route only when parity-safe

## Contract parity reminders

- Keep path and method shape.
- Keep status codes, headers, and error behavior according to selected migration mode.
- Keep pagination/filtering/searching envelope and query semantics.
- Keep parser/renderer behavior for declared media types.
- Keep `204` responses body-less.
- DRF `partial=True` update semantics -> Schema with `Optional` fields for partial updates.
- Verify controller-generated route names, prefixes, and trailing slash behavior.
- Verify `ModelController` defaults before adopting generated CRUD.
- Verify async auth only with async handlers.
- In async handlers/controllers, do not return unevaluated sync `QuerySet` objects; use async ORM methods where available or wrap sync ORM access with `sync_to_async(..., thread_sensitive=True)`.
- For django-ninja 1.6+, use `Status(status_code, body)` instead of deprecated `(status_code, body)` tuple returns.

## Validation reminders

- Verify installed compatibility before migration: Django `>=6.0`, Python `>=3.12`, django-ninja `>=1.6.2`, and django-ninja-extra `>=0.31.4` when using controllers/model controllers.
- Run `uv run python -Wd manage.py check` (or `docker exec <web-container> python -Wd manage.py check`) before editing and after each migrated slice.
- Run repository-native linters and tests after each migration slice.
- Prefer existing integration tests as the contract source of truth.
- Treat generated controllers/schemas/helpers as implementation conveniences, not as proof of parity.
