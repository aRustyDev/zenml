# 0008 — Service Connectors

Repo @ `cc064f100`. Source root `src/zenml/`. Framework reference: the six nullable dotted-path
gates on `ServerConfiguration` (`src/zenml/config/server_config.py:343-349`) and the three-tier
model established in the audit preamble.

## 1. Surface

| Layer | Path |
|---|---|
| Core abstraction | `src/zenml/service_connectors/service_connector.py` (`ServiceConnectorMeta` metaclass, `ServiceConnector` at `:110`) |
| Auto-registration | `src/zenml/service_connectors/service_connector.py:104` — `service_connector_registry.register_service_connector_type(cls.get_type())` fires from the metaclass |
| Registry | `src/zenml/service_connectors/service_connector_registry.py` (`ServiceConnectorRegistry:31`, `register_service_connector_type:40`, `instantiate_connector:165`, `register_builtin_service_connectors:205`) |
| Full-stack helper | `src/zenml/service_connectors/service_connector_utils.py:166` (`get_resources_options_from_resource_model_for_full_stack`), `_raise_specific_cloud_exception_if_needed:58` |
| Built-in connectors | `src/zenml/service_connectors/docker_service_connector.py`; `src/zenml/service_connectors/oauth2_service_connector.py:211` (`OAuth2ServiceConnector`, spec at `:149`) |
| Integration connectors | `integrations/{aws,azure,gcp,kubernetes,hyperai}/service_connectors/` — 5 packages |
| Type descriptor models | `src/zenml/models/v2/misc/service_connector_type.py` (`ResourceTypeModel:35`, `AuthenticationMethodModel:89`, `ServiceConnectorTypeModel:219`, `ServiceConnectorRequirements:473`, `ServiceConnectorResourcesModel:574`) |
| Domain model | `src/zenml/models/v2/core/service_connector.py` |
| SQL schema | `src/zenml/zen_stores/schemas/service_connector_schemas.py` (`from_request:155`, `update:199`) |
| REST router | `src/zenml/zen_server/routers/service_connectors_endpoints.py` (`router:69`, `types_router:75`) |
| CLI | `src/zenml/cli/service_connectors.py` — 2221 lines, the largest CLI module in the repo (`service_connector:48`, `register_service_connector:513`, `list_service_connectors:989`) |
| Migrations | `zen_stores/migrations/versions/0b06faa59c93_add_service_connectors.py`, `e5225281b4d3_add_connector_skew_tolerance.py`, `5bb25e95849c_add_internal_secrets.py` |

## 2. Tier verdict — **Tier B**, with a Tier-C authorization overlay

**Tier B.** The whole connector stack is implemented in OSS. The one capability off by default
is *implicit authentication*, and its implementation is present and concrete:
`ServiceConnector._check_implicit_auth_method_allowed` (`service_connector.py:516`) with the
gate at `service_connector.py:527`. Flipping `ZENML_ENABLE_IMPLICIT_AUTH_METHODS`
(`constants.py:242`) enables it — no Pro artefact is involved.

Concrete connector classes shipping in OSS:

| Class | File |
|---|---|
| `DockerServiceConnector` | `service_connectors/docker_service_connector.py` |
| `OAuth2ServiceConnector` | `service_connectors/oauth2_service_connector.py:211` |
| `AWSServiceConnector` | `integrations/aws/service_connectors/aws_service_connector.py` |
| `GCPServiceConnector` | `integrations/gcp/service_connectors/gcp_service_connector.py` |
| `AzureServiceConnector` | `integrations/azure/service_connectors/azure_service_connector.py` |
| `KubernetesServiceConnector` | `integrations/kubernetes/service_connectors/kubernetes_service_connector.py` |
| `HyperAIServiceConnector` | `integrations/hyperai/service_connectors/hyperai_service_connector.py` |

**No Pro-only connector class exists in-tree.** Verified two ways: (a) the complete import list
in `register_builtin_service_connectors` (`service_connector_registry.py:205-269`) names exactly
the seven above; (b) `grep -rn "register_service_connector_type("` across `src/zenml/` returns
only the metaclass call (`service_connector.py:104`) and the registry definition
(`service_connector_registry.py:40`) — there is no other registration hook a Pro package
could target without monkey-patching.

**Tier-C overlay.** Every router path calls RBAC
(`service_connectors_endpoints.py:109,150,250,279,301,333,372,409,410,503,507`), and
`ResourceType.SERVICE_CONNECTOR = "service_connector"` (`rbac/models.py:71`) is **not**
project-scoped — it is in the exclusion list at `rbac/models.py:92`. In OSS every one of those
checks early-returns (`rbac/utils.py:75,110,243,272,316,381`), so connectors are globally
visible and mutable by any authenticated user. `SERVICE_CONNECTOR` is absent from
`DEFAULT_REPORTABLE_RESOURCES` (`constants.py:465`), so it is unmetered on Pro as well.

## 3. Gate inventory

| # | Mechanism | Location | Effect |
|---|---|---|---|
| 2 | Env-var boolean | `ENV_ZENML_ENABLE_IMPLICIT_AUTH_METHODS` (`constants.py:242`) read through `handle_bool_env_var` (`constants.py:131`) at `service_connector.py:527`; per-instance escape `allow_implicit_auth_methods` (`service_connector.py:158`) checked first at `:525` | **Default DENY.** `AuthorizationException` (`service_connector.py:528-534`) → HTTP 401 (`zen_server/exceptions.py:77`). Enforced at `aws_service_connector.py:900`, `gcp_service_connector.py:1194,1225`, `azure_service_connector.py:646` only |
| 6 | Helm value | `helm/values.yaml:328` (`zenml.enableImplicitAuthMethods: false`) → `helm/templates/_environment.tpl:671-672` | Same gate as chart config; default `false` |
| 1 | Dependency presence | `register_builtin_service_connectors` wraps every import in `try/except ImportError` logging only at debug level (`service_connector_registry.py:218-219,224-225,231-232,238-240,247-249,256-257,266-268`) | A connector with a missing extra is **silently absent** from `list-types` — no error, no warning |
| 1 | Dependency presence, client↔server split | `ServiceConnectorTypeModel.local:279` / `.remote:283`; reconciled in `rest_zen_store.py:3426-3427,3439-3441,3698,3734-3739` | Type installed server-side but not client-side → `local=False, remote=True`; usable through the server, not locally |
| 3 | Pluggable implementation source | `SecretsStoreType` (`enums.py:271-281`) → `BaseSecretsStore.get_store_class` (`base_secrets_store.py:197-250`) and `convert_config` (`:65-137`), incl. `_load_custom_store_class:165` | Selects where connector credentials are actually persisted |
| 8 | Schema availability | `0b06faa59c93_add_service_connectors.py`; migrations skipped when `ZENML_DISABLE_DATABASE_MIGRATION` is set (`sql_zen_store.py:1551-1557`) | Router fails against an un-migrated DB |
| 4 | Entitlement gate | **Absent.** `service_connectors_endpoints.py` imports no `check_entitlement`/`report_usage`; `SERVICE_CONNECTOR` not in `DEFAULT_REPORTABLE_RESOURCES` (`constants.py:465`) | Unlimited connectors on both editions |
| 5 | Deployment-type gate | **Absent** connector-specifically; only the blanket `is_pro_server` auth override (`server_config.py:851-885`) applies |  |
| 7 | Three-state `Optional[bool]` cascade | **Absent** |  |
| 9 | Deprecation | `workspace_router` imported from `projects_endpoints.py` (`service_connectors_endpoints.py:62`), mounted at `:88,121,187`, each flagged `deprecated=True` (`:91,124,190`) with the comment "kept for dashboard compatibility and can be removed after the migration" (`:86-87`) | Legacy `/workspaces/{project}/service_connectors` paths still answer, and are correctly marked deprecated in the OpenAPI schema. No `@experimental` decorator exists anywhere in the repo |

## 4. OSS degradation

| Capability | OSS behaviour | Substitute |
|---|---|---|
| Implicit auth (EC2 instance profile, GCP ADC, Azure managed identity, local `~/.aws` profile) | Disabled by default; `AuthorizationException` at `service_connector.py:528` | `ZENML_ENABLE_IMPLICIT_AUTH_METHODS=true`, or explicit secret-based auth methods |
| Per-connector authorization | Every check fails open (`rbac/utils.py:75,110,243,272,316,381`) | `verify_admin_status_if_no_rbac` (`zen_server/utils.py:793`) is the OSS substitute elsewhere — but **the connector router never calls it**, so OSS has *no* authorization on connectors at all |
| `Action.CLIENT` (`rbac/models.py:39`) — minting short-lived downstream credentials | `verify_permission_for_model(connector, action=Action.CLIENT)` at `service_connectors_endpoints.py:410` fails open | none |
| Sharing / resource membership | `update_resource_membership` and the delete paths early-return (`rbac/utils.py:812,839,860`) | Everything is effectively shared |
| Connector-type availability | Governed purely by installed extras: `connectors-kubernetes`, `connectors-aws`, `connectors-gcp`, `connectors-azure` (`pyproject.toml:118-136`) | `zenml integration install <name>` |
| Credential storage | **Not degraded.** All five secrets-store backends ship in OSS | — |

### The credentials are not in the connector row

`ServiceConnectorSchema.from_request` persists only `connector_request.configuration.non_secrets`
base64-encoded (`service_connector_schemas.py:172,184-188`) plus a `secret_id` FK (`:190`);
`update` explicitly excludes `{"user", "secrets"}` (`service_connector_schemas.py:216`). The
secret body is written by `SqlZenStore._create_connector_secret`
(`sql_zen_store.py:10945`, called at `:10590` and `:11093`) and rotated at `:10880`. Therefore:

- The **secrets store is a hard dependency** of any credential-bearing connector. Backends:
  `sql`/`aws`/`gcp`/`azure`/`hashicorp`/`custom`/`none` (`enums.py:274-280`), concrete classes at
  `zen_stores/secrets_stores/{sql,aws,gcp,azure,hashicorp}_secrets_store.py` — **all OSS**.
- `SecretsStoreType.NONE` disables storage entirely (`sql_zen_store.py:1612,1630`;
  `base_zen_store.py:400`; reported in `models/v2/misc/server_models.py:78`). A server configured
  that way cannot hold connector credentials at all.
- **Bootstrap cycle:** `ServiceConnectorSecretsStore`
  (`zen_stores/secrets_stores/service_connector_secrets_store.py`) uses a service connector to
  reach the cloud secret manager, and force-sets `allow_implicit_auth_methods = True` on its own
  connector (`:144-148`) — the only in-tree caller that bypasses gate #2.

## 5. Self-host path (Tier B)

```
ZENML_ENABLE_IMPLICIT_AUTH_METHODS=true     # constants.py:242 → service_connector.py:527
# Helm equivalent:
zenml.enableImplicitAuthMethods: true       # helm/values.yaml:328 → _environment.tpl:671-672
```

Classes thereby enabled: `AWSServiceConnector` (`aws_service_connector.py:900`),
`GCPServiceConnector` (`gcp_service_connector.py:1194,1225`), `AzureServiceConnector`
(`azure_service_connector.py:646`). Kubernetes, Docker and OAuth2 connectors never consult
the gate and so are unaffected.

There is **no Tier-A gap** for connectors — nothing has to be written to reach parity.
Optional hardening, all OSS: the `secrets-aws` / `secrets-gcp` / `secrets-azure` /
`secrets-hashicorp` extras (`pyproject.toml:111-114`).

## 6. Blast radius

| Direction | Target |
|---|---|
| Registration (upstream) | `ServiceConnectorMeta.__new__` fires at **class-definition time** (`service_connector.py:104`) — importing a module registers its type. This is why `register_builtin_service_connectors` is nothing but bare imports |
| Consumers (downstream) | Stack components match connectors via `ServiceConnectorRequirements.is_satisfied_by` (`models/v2/misc/service_connector_type.py:493`) — artifact stores, container registries, orchestrators, image builders |
| Secrets store (lateral) | `_create_connector_secret` / `_update_connector_secret` (`sql_zen_store.py:10945`, `:10880`); read-back at `:10660-10662`. Every connector CRUD is a two-phase write across the DB row and the secrets backend — a partial failure leaves an orphan |
| Reverse hop | `ServiceConnectorSecretsStore:144-148` consumes a connector to back the secrets store |
| Client/server reconciliation | `rest_zen_store.py:3426-3441` (get), `:3698` (list types), `:3734-3739` (type merge) |
| Full-stack wizard | `service_connector_utils.py:166` → `service_connectors_endpoints.py:474` (`get_resources_based_on_service_connector_info`), which **does** call `verify_permission` twice (`:503,:507`) |
| RBAC injection | `RBACSqlZenStore` (`zen_server/rbac/rbac_sql_zen_store.py:47`) is selected whenever `ENV_ZENML_SERVER` is set (`base_zen_store.py:157-162`) — i.e. on OSS servers too; it adds no connector-specific override |

## 7. Test coverage

No `tests/unit/service_connectors/` directory exists, and no
`tests/unit/zen_stores/secrets_stores/` either — both `ls` calls error. Adjacent coverage:

| Path | Covers |
|---|---|
| `tests/integration/functional/zen_stores/test_secrets_store.py` | The secrets-store backends connectors depend on |
| `tests/integration/functional/zen_stores/test_zen_store.py` | Connector CRUD through the store |
| `tests/unit/integrations/gcp/`, `tests/unit/integrations/kubernetes/` | Integration-local helpers, not the connector base class |

**Gap:** `_check_implicit_auth_method_allowed` (`service_connector.py:516`) — the only
security-relevant gate in this area — has no unit test. Nor does the
`allow_implicit_auth_methods=True` bypass used by `ServiceConnectorSecretsStore:148`. A
regression that inverted the default would not be caught by `tests/unit/`.
