# 0003 — User Management & Authentication

Repo `cc064f100`. All paths relative to repo root. Every claim carries `file:line`.

---

## 1. Surface

| Layer | Path | Size |
|---|---|---|
| Auth core | `src/zenml/zen_server/auth.py` | 1,418 |
| JWT | `src/zenml/zen_server/jwt.py` — `JWTToken` (`:34`), `decode_token` (`:68`), `encode` (`:220`) | 268 |
| Routers | `routers/users_endpoints.py` (774), `service_accounts_endpoints.py` (517), `devices_endpoints.py` (304), `auth_endpoints.py` (686) | 2,281 |
| Pydantic models | `models/v2/core/user.py` (558), `service_account.py` (322), `api_key.py` (452), `device.py` (484) | 1,816 |
| SQL schemas | `zen_stores/schemas/user_schemas.py` (350), `api_key_schemas.py` (277), `device_schemas.py` (315) | 942 |
| CLI | `cli/user_management.py` (446, 7 cmds from `:44`), `cli/service_accounts.py` (590, two groups — `service_account` `:94` + `api_key` `:307`), `cli/authorized_device.py` (159, 5 cmds from `:37`), `cli/login.py` (1,357) | 2,552 |
| Pro client (client-side) | `src/zenml/login/pro/` — `client.py` (512), `organization/` (125), `workspace/` (281), `utils.py` (106), `constants.py`/`models.py`; plus `login/credentials_store.py` (670), `login/web_login.py` (270) | 2,033 |
| Enum | `enums.py:341` `AuthScheme` (members `:344-347`); `enums.py:350` `OAuthGrantTypes`; `enums.py:359` `OAuthDeviceStatus` | — |
| Store info | `models/v2/misc/server_models.py:81` publishes `auth_scheme` to clients; set at `zen_stores/base_zen_store.py:398,411` | — |

No migration is specific to user management — `UserSchema` predates the audited boundary (`user_schemas.py:74-92`).

---

## 2. Tier verdict — **Tier C, with a Tier-A identity provider**

Not Tier A (no `*Interface` ABC is involved) and not Tier B (nothing to switch on). The gate here is mechanism **5 + 7**: `AuthScheme` selects a *code path*, and `EXTERNAL` **removes** endpoints rather than adding an implementation.

Concrete evidence:

- `authentication_provider()` (`auth.py:1354-1373`) dispatches on `server_config().auth_scheme` (`:1363`). `EXTERNAL` reuses `oauth2_authentication` (`:1370-1371`) — same transport, different credential source.
- The identity provider itself is **external, not an in-tree class**: `authenticate_external_user()` (`auth.py:740`) `assert`s `config.external_user_info_url is not None` (`:759`) and HTTP-GETs it (`:770-775`) with `server_id` as a query param (`:768`). There is no OSS implementation because there is no interface to implement — the "implementation" is a remote HTTP service whose URL `get_server_config()` hard-codes to `f"{server_pro_config.api_url}/users/authorize_server"` (`server_config.py:861-863`).
- Fail-open is the RBAC one (dossier 0001): with no RBAC, `verify_admin_status_if_no_rbac` (`zen_server/utils.py:793-822`) degrades authorization to `UserSchema.is_admin` (`user_schemas.py:92`).

So: **OSS gets full local user management with a two-valued permission model; Pro gets a federated identity provider and loses local user management entirely.** Unusually for this audit, the OSS side has *more* endpoints.

---

## 3. Gate inventory

| # | Mechanism | Where |
|---|---|---|
| **5** | Deployment-type gate | `is_pro_server` (`server_config.py:820-826`) → `get_server_config()` force-overwrites `auth_scheme = AuthScheme.EXTERNAL` (`:857`), `external_login_url` (`:858-860`), `external_user_info_url` (`:861-863`), `external_server_id = workspace_id` (`:864`). **The operator cannot opt out**: the assignment is unconditional inside the `is_pro_server` branch (`:851-885`) |
| **7** | Three-state cascade → endpoint removal | Five module-level `if` blocks strip routes at import time: `users_endpoints.py:137` (create user), `:228` (activate/deactivate/delete/email-opt-in — `:239,299,347,389`), `:617` (`PUT /current-user`); `server_endpoints.py:224` (server activation block); `zen_server_api.py:360` (the whole `users_endpoints.activation_router`). All spelled `auth_scheme != AuthScheme.EXTERNAL` |
| **5b** | Inline EXTERNAL guards | `users_endpoints.py:465-469` — under EXTERNAL, `update_user` is narrowed to `default_project_id` only. `auth.py:325-335` — a local (non-external, non-service) account is refused under EXTERNAL. `auth.py:1015-1021` — local service-account API keys under EXTERNAL log a deprecation warning but still work. `sql_zen_store.py:13847,13882` — server activation paths |
| **5c** | Pro-only hard block | `service_accounts_endpoints.py:69-81` `_ensure_workspace_service_account_mutation_allowed()`; the condition is at **`:76`** (`auth_scheme == AuthScheme.EXTERNAL`) and it raises `IllegalOperationError` → **403** (`zen_server/exceptions.py:74`) |
| **5d** | Device flow disabled | `auth.py:358-366` (device-bound token rejected under `NO_AUTH`/`EXTERNAL`), `auth_endpoints.py:180-185` (`OAUTH_DEVICE_CODE` grant → 403), `auth_endpoints.py:381-386` (`/device_authorization` → 403). `ZENML_EXTERNAL` grant is the mirror image: rejected unless `EXTERNAL` (`auth_endpoints.py:211-217`) |
| **3** | Pluggable implementation source | **not used for auth.** No `auth_implementation_source`. `AuthScheme` is a closed enum (`enums.py:341-347`) and `authentication_provider()` raises `ValueError` on anything else (`auth.py:1372-1373`) — this is the one audited area with **no** plug-in seam |
| **4** | Entitlement gate | none on user/auth endpoints. `grep` for `check_entitlement` returns no hits in `users_endpoints.py`, `service_accounts_endpoints.py`, `devices_endpoints.py`, `auth_endpoints.py` — **seats are not metered through the OSS feature gate** |
| **6** | Helm value | `zenml.pro.enabled` (`helm/values.yaml:86,91`) is the master switch; `zenml.auth.externalServerID` (`:290`) is documented as "overridden if `zenml.pro.enabled` is set" (`:288-289`) |
| **2** | Env-var | `ZENML_SERVER_AUTH_SCHEME` via the generic prefix loop (`server_config.py:834-841`), but on a Pro server `:857` overwrites whatever was passed. Pro-side config comes from the separate `ENV_ZENML_SERVER_PRO_PREFIX` block, deliberately skipped by the generic loop (`:838-841`) and re-read by `ServerProConfiguration` (`:926-951`) |
| **8** | Schema availability | n/a — `user`/`api_key`/`device` tables ship in every deployment |
| **9** | Deprecation | `auth.py:1015-1021` and `service_accounts_endpoints.py:77-81` both describe workspace-local service accounts as **deprecated** in favour of Pro organization service accounts. No experimental/deprecation decorator exists; it is prose plus a raise |
| — | RBAC coupling | `users_endpoints.py:696` registers `POST /users/resource_membership` only when `rbac_enabled`. Six RBAC calls are commented out: `:116-123`, `:177-181`, `:218-222`, `:322-326`, `:373-378`, `:508-512`. `ResourceType.USER` itself is commented out (`rbac/models.py:79`, `:101`, `rbac/utils.py:753`) |

---

## 4. OSS degradation

### What OSS *keeps* (and Pro loses)

| Capability | OSS | Pro (`EXTERNAL`) |
|---|---|---|
| `POST /users` | ✅ | removed (`users_endpoints.py:137`) |
| activate / deactivate / delete user | ✅ | removed (`:228`) |
| `PUT /current-user` | ✅ | removed (`:617`) |
| `PUT /users/{id}` | full | narrowed to `default_project_id` (`:465-469`) |
| activation router | registered | not registered (`zen_server_api.py:360`) |
| device authorization (`zenml login` browser flow) | ✅ | 403 (`auth_endpoints.py:381-386`) |
| create/update/delete service accounts + API keys | ✅ | 403 (`service_accounts_endpoints.py:76`) |
| `zenml user create/delete` CLI | works | server 404/403 |

### What OSS *loses*

- **SSO / federated identity.** No OIDC, SAML or external IdP path exists without `EXTERNAL`, and `EXTERNAL` requires `external_user_info_url` (`auth.py:759`), which `get_server_config()` only ever points at the Pro API (`server_config.py:862`). Setting it by hand to a third-party endpoint is possible in principle (`ZENML_SERVER_EXTERNAL_USER_INFO_URL`, `ZENML_SERVER_EXTERNAL_LOGIN_URL`, `ZENML_SERVER_EXTERNAL_SERVER_ID`) — the contract is just `ExternalUserModel.model_validate(payload)` (`auth.py:799`) — but the moment `EXTERNAL` is set, every local user-management endpoint above disappears.
- **Teams / groups.** No `TeamSchema` or `TeamResponse` exists in `src/zenml` (only historical migrations, e.g. `migrations/versions/alembic_start.py:174-202`). The `team_id` parameter on the sharing endpoint (`users_endpoints.py:712`) is inert.
- **Roles.** Nothing between "admin" and "not admin".

**Substitute: the `is_admin` boolean.** `UserSchema.is_admin` (`user_schemas.py:92`, default `False`), surfaced at `models/v2/core/user.py:172,290,433-439`, enforced at 14 call sites (see dossier 0001 §4) — 5 in `users_endpoints.py`, 7 in `service_accounts_endpoints.py`, 2 in `secrets_endpoints.py`, plus `server_endpoints.py:207-218`.

### ⚠️ Sharpest consequence: service accounts are permanently non-admin

`UserSchema.from_service_account_request` hardcodes `is_admin=False` (`user_schemas.py:215`), and `ServiceAccountResponse.to_user_model()` does the same (`models/v2/core/service_account.py:219`). API-key authentication builds the auth context from exactly that conversion (`auth.py:1038-1039`).

So on an **OSS server with RBAC off**, an API-key caller can never satisfy `verify_admin_status_if_no_rbac` (`zen_server/utils.py:816-820`). That blocks, for automation:

- `POST /users` (`users_endpoints.py:173`), deactivate (`:319`), delete (`:370`), update-other-user (`:505`)
- create/update/delete service accounts and every API-key mutation (`service_accounts_endpoints.py:113,219,257,315,429,480,510`)
- secrets backup/restore (`secrets_endpoints.py:268,302`)

There is no flag to make a service account an admin — the field is not on `ServiceAccountRequest`. **CI/CD cannot manage identities on an OSS ZenML server; only an interactive human account can.** Turning RBAC on (Pro) is what lifts this, because `:810` then skips the check entirely.

---

## 5. Self-host path

Nothing to enable via a class + env var — no ABC, no source field. Three options, in order of cost:

1. **Stay on OSS auth, live with `is_admin`.** Free. Set `ZENML_SERVER_AUTH_SCHEME=OAUTH2_PASSWORD_BEARER` (the default path; `auth.py:1368-1369`) and manage users with `zenml user …` (`cli/user_management.py:39-44`).
2. **Point `EXTERNAL` at your own IdP shim.** Set `ZENML_SERVER_AUTH_SCHEME=EXTERNAL`, `ZENML_SERVER_EXTERNAL_LOGIN_URL`, `ZENML_SERVER_EXTERNAL_USER_INFO_URL`, `ZENML_SERVER_EXTERNAL_SERVER_ID` (fields at `server_config.py:339-341`). The shim must accept `GET ?server_id=<uuid>` with `Authorization: Bearer <token>` (`auth.py:765-775`) and return JSON validating as `ExternalUserModel` (`auth.py:799`). Small job — a few hundred lines of proxy. **But the cost is severe**: you inherit all five endpoint-removal blocks (§3 row 7), so you can no longer create, activate, deactivate or delete users through ZenML, the device-code login flow dies (`auth_endpoints.py:381-386`), and workspace service accounts become un-mutable (`service_accounts_endpoints.py:76`). Everything must be provisioned by your shim. Note this does **not** require `deployment_type=CLOUD`, so no Pro plumbing is dragged in.
3. **Roles beyond `is_admin`** — requires the whole of dossier 0001 §5 (an RBAC backend), since `is_admin` checks are disabled the instant `rbac_enabled` is true.

---

## 6. Blast radius

**`auth.py` is the widest fan-in surface in the server.** `authorize` / `Security(authorize)` is a dependency on essentially every router; `authentication_provider()` (`:1354`) is resolved once at import.

Hops that matter:

1. **`server_config.py:851-885` is a single mutation point for four subsystems at once** — auth scheme, RBAC source, feature-gate source, and dashboard URL. A change to `is_pro_server` (`:820`) simultaneously alters authentication, authorization, metering and endpoint registration. It is the highest-leverage 35 lines in the repo.
2. **Import-time route registration.** The five `if server_config().auth_scheme != AuthScheme.EXTERNAL:` blocks run at *module import*, so the OpenAPI surface is fixed at process start. Changing `ZENML_SERVER_AUTH_SCHEME` requires a restart, and a test that monkeypatches `server_config()` after import will not move these routes.
3. **Client-visible coupling.** `auth_scheme` is published in `ServerModel` (`models/v2/misc/server_models.py:81`, populated `base_zen_store.py:398,411`) and read by `orchestrators/utils.py:144` — orchestrator code branches on the server's auth scheme.
4. **Token→identity fan-out.** `authenticate_credentials` (`auth.py:184`) resolves, in one function: password tokens, API-key tokens (`:1004` detects the `ZENML_PRO_API_KEY_PREFIX`, `:46`), device tokens (`:358`), and external tokens (`:1008-1013`). It also re-verifies API-key generation (`:349-355`) and password-change recency (`:337-346`).
5. **Service-account ↔ user conflation.** `to_user_model()` (`service_account.py:193`) exists because "a lot of code still relies on the active user in the auth context being a `UserResponse`" (`auth.py:1034-1037`). This is why `is_admin=False` leaks into every downstream admin check (§4).
6. **RBAC's Pro-compat carve-out.** `ZenMLCloudRBAC.check_permissions` grants *everything* to a workspace-local service account with no `external_user_id` (`rbac/zenml_cloud_rbac.py:57-61`) — on Pro, a legacy API key bypasses RBAC entirely. Paired with the deprecation at `auth.py:1015-1021`.
7. **Vocabulary trap.** `projects_endpoints.py:62-63` still serves a router at prefix `WORKSPACES` with `deprecated=True` on every route (`:78,115,144,177,210,240`) — the pre-rename Project. A ZenML **Pro workspace** is a different thing (`server_config.py:864,870,875`). Grepping "workspace" in this area returns both.

---

## 7. Test coverage

There is **no** `tests/unit/zen_server/rbac/` and **no** `tests/unit/zen_server/routers/`. Relevant files:

| File | Lines | Covers |
|---|---|---|
| `tests/unit/zen_server/test_auth.py` | 500 | authentication paths |
| `tests/unit/zen_server/test_jwt.py` | 301 | token encode/decode |
| `tests/unit/conftest.py` | — | one `is_service_account=False` fixture field (`:358`) |
| `tests/unit/zen_server/endpoints/` | — | only `test_webhook_endpoints.py` |
| `tests/unit/cli/` | — | 3 modules, **none** for `user_management`, `service_accounts`, `authorized_device` or `login` |

`grep -rln "service_account|api_key|authorized_device" tests/unit --include="*.py"` → 3 files (`conftest.py`, `test_auth.py`, `test_jwt.py`).

**Best-tested area in this audit** (801 lines of unit tests vs. RBAC's effectively zero and resource pools' 2,020 skipped) — but the coverage is on *authentication*, not on the gating. No test asserts that the `EXTERNAL` endpoint-removal blocks remove the right routes, that `verify_admin_status_if_no_rbac` denies a service account, or that `is_admin=False` is forced on service-account creation. The ~2,552 lines of identity CLI have no unit tests at all.
