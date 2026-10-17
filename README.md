# Medi Car BD UI-aligned Flutter app boilerplate

Each mobile presentation project (`mcb-user-app` and `mcb-driver-app`) is bootstrapped from `mcb-app-boilerplate` and communicates strictly with its dedicated backend client through `mcb-api-gateway`. A mobile app owns the presentation state, local cache, device hardware integrations (GPS, push notifications, secure keystores) and real-time socket subscriptions required by exactly one persona. It must never call another client's route prefix directly; shared authentication flows are reached through `/api/v1/auth/*` on the gateway.

## Mobile UI-to-client mapping

| Presentation project | Dedicated backend client | Gateway prefix | Primary responsibility |
| --- | --- | --- | --- |
| `mcb-user-app` | `mcb-user-client` | `/api/v1/user` | Customer identity/profile, booking, live trip view, payment history and notifications |
| `mcb-driver-app` | `mcb-driver-client` | `/api/v1/driver` | Driver availability, assignments, trip actions, background location updates and SOS |

Web presentation projects (`mcb-dispatch-ui`, `mcb-dashboard-ui`, `mcb-org-ui`, `mcb-hrm-ui`, `mcb-billing-ui`) are derived from `mcb-ui-boilerplate` instead of this repository.

## Shared backend dependencies

- `mcb-api-gateway` - single edge entry point for REST proxying, rate limits, `x-correlation-id` propagation and WebSocket (`ws: true`) upgrades
- `mcb-auth-service` - sign-in, OTP verification, session management, JWT issuance and token rotation (`/api/v1/auth/*`)
- `mcb-notification` - dispatches FCM push and SMS notifications triggered by domain events over Kafka

## Architecture and module layout

The Flutter project enforces Clean Architecture with a feature-first module layout. Domain entities and use cases have zero dependencies on Flutter widgets, HTTP clients or local storage plugins.

| Layer / Directory | Meaning | Examples |
| --- | --- | --- |
| `lib/core/config` | Environment validation and runtime flavor configuration | `env_config.dart`, `app_flavor.dart` |
| `lib/core/network` | Dio REST client, interceptors and Socket.IO manager | `api_client.dart`, `auth_interceptor.dart`, `socket_service.dart` |
| `lib/core/services` | Hardware-backed token storage, geolocator and FCM | `secure_storage_service.dart`, `location_service.dart` |
| `lib/core/theme` | Material Expressive design tokens, light/dark themes and Ubuntu typography | `app_theme.dart`, `app_colors.dart` |
| `lib/features/<name>/data` | DTOs, remote/local data sources and repository implementations | `auth_remote_datasource.dart`, `user_model.dart` |
| `lib/features/<name>/domain` | Pure Dart entities, repository contracts and use cases | `user_entity.dart`, `auth_repository.dart` |
| `lib/features/<name>/presentation` | State controllers/notifiers, screens and feature-scoped widgets | `login_screen.dart`, `auth_controller.dart` |

Cross-feature imports between presentation or data layers are prohibited; shared state is exposed exclusively through `lib/core/` or domain contracts.

## Bootstrapping and running the app

Clone or generate a new repository from this template, rename the bundle identifier for the target persona, and supply the runtime configuration.

```bash
cp .env.example .env        # fill in APP_ROLE, GATEWAY_BASE_URL, GATEWAY_WS_URL, CLIENT_API_PREFIX
flutter pub get
```

There is deliberately no hardcoded fallback URL in Dart code: the app refuses to boot if `GATEWAY_BASE_URL`, `GATEWAY_WS_URL` or `CLIENT_API_PREFIX` is missing from the environment configuration.

Rename the package identifier before building `mcb-user-app` or `mcb-driver-app`:

```bash
dart run change_app_package_name:main com.medicarbd.user
```

When testing against a local `mcb-infra` Docker Compose stack, point `GATEWAY_BASE_URL` to the host loopback (`http://10.0.2.2:8000` on Android emulators, `http://localhost:8000` on iOS simulators) or pass configuration via `--dart-define-from-file`:

```bash
flutter run --dart-define-from-file=.env
```

Production release artifacts are compiled with obfuscation and split debug symbols:

```bash
flutter build appbundle --release --obfuscate --split-debug-info=build/app/outputs/symbols
```

## `mcb-user-app` versus `mcb-driver-app`

Although both mobile apps originate from `mcb-app-boilerplate`, they must remain separate binaries and repositories. They operate under different OS permission models, battery constraints and RBAC roles.

| Concern | `mcb-user-app` | `mcb-driver-app` |
| --- | --- | --- |
| Target caller role | `USER` | `DRIVER` |
| Dedicated upstream client | `mcb-user-client` (`/api/v1/user/*`) | `mcb-driver-client` (`/api/v1/driver/*`) |
| Registration flow | Self-registration via public `POST /api/v1/auth/register` | Provisioned by `ORG_ADMIN` or `HR_MANAGER` via `POST /api/v1/auth/user` |
| Location tracking | Foreground-only while selecting pickup or viewing active trip | Foreground service + background isolate streaming GPS telemetry |
| Socket.IO channels | Subscribes to trip status, driver ETA and invoice events | Publishes live coordinates, receives dispatch offers and triggers SOS |
| Wake-lock and battery profile | Standard mobile lifecycle; suspends in background | Persistent notification and wake-lock while availability is online |

## Gateway-aligned network and WebSocket implementation

The mobile network stack is designed around `mcb-api-gateway`'s exact proxy behavior. REST calls use `Dio` with a queued interceptor pipeline, and real-time streams use `socket_io_client` over the gateway's WebSocket upgrade path.

| Capability | Implementation |
| --- | --- |
| Base routing | Single `GATEWAY_BASE_URL`; client paths resolve under `CLIENT_API_PREFIX` (`/api/v1/user` or `/api/v1/driver`) |
| Token storage | `flutter_secure_storage` using Android EncryptedSharedPreferences/Keystore and iOS Keychain |
| Token refresh | `QueuedInterceptorsWrapper` catches `401`, locks the queue, calls `POST /api/v1/auth/refresh` and retries |
| Correlation IDs | Generates a v4 UUID in `x-correlation-id` per outgoing request and Socket.IO handshake for distributed tracing |
| WebSockets | `socket_io_client` forced to `['websocket']` transport so `mcb-api-gateway`'s `upgrade` handler proxies cleanly |
| Upstream failure handling | Maps gateway `502` (dead/slow upstream) and `503` (disabled route) to non-destructive retryable UI states |
| Rate-limit handling | Respects `429` responses and `Retry-After` headers without logging out the user |

Four client-side rules are worth recording, because violating any of them breaks gateway compatibility or security:

- **The app never sends custom identity or role headers.** `mcb-api-gateway` does not inject or trust headers like `x-user-role` or `x-tenant-id`. The mobile app sends only `Authorization: Bearer <token>`, which both the gateway and the upstream `-client` verify via `mcb-shared-auth`.
- **Content-Type headers are always explicit on mutation requests.** Because `mcb-api-gateway` never parses request bodies (to avoid draining the stream before `http-proxy-middleware` forwards it), the mobile client must send well-formed JSON payloads with explicit `Content-Type: application/json` headers on `POST`, `PUT` and `PATCH`.
- **Retries use exponential backoff with jitter.** Field apps operate on mobile networks behind carrier-grade NAT (CGNAT) where multiple callers share a public IP. Aggressive instant retries from a mobile client will trip the gateway's per-IP rate limiter for other users on the same carrier tower.
- **WebSocket connections authenticate on handshake and reconnect on token rotation.** When `AuthInterceptor` rotates an expired access token via `mcb-auth-service`, it pushes the new JWT to the active Socket.IO instance so subsequent reconnections do not fail gateway validation.

### Known security consideration on the client

Because `mcb-shared-auth` currently verifies JWT signature and expiry without checking `audience` or `issuer`, the mobile client must still validate the decoded JWT `role` claim locally before mounting role-specific navigation trees. If a `DRIVER` account signs into `mcb-user-app` (or vice versa), the client-side auth guard immediately revokes the local session and clears secure storage rather than waiting for upstream `403` rejections.

## RBAC and tenant scoping on mobile

Public sign-up inside `mcb-user-app` only ever creates a plain `USER`. Drivers cannot self-register from `mcb-driver-app`; driver accounts are provisioned within a tenant by `ORG_ADMIN` or `HR_MANAGER` and sign in using issued credentials or OTP.

Upon authentication, the mobile client decodes the non-sensitive payload of the JWT (`sub`, `role`, `tenantId`) to scope local Hive/SQLite caches by `tenantId` and user ID. All local cache tables are wiped on sign-out or tenant switch to prevent cross-account data leakage on shared field devices.
