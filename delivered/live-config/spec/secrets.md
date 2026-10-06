# Secret store

Credentials the hub holds for the records that need them: written once, never returned, encrypted at rest, and referred
to by name.

## The record

`secrets` holds one row per secret.

| Column        | Type        | Meaning                                                                        |
| ------------- | ----------- | ------------------------------------------------------------------------------ |
| `name`        | String, PK  | A slug — lowercase letters, digits, hyphens. Immutable.                        |
| `ciphertext`  | Text        | Base64 of the AES-256-GCM output (`bzh:sql-portable`: no backend binary type). |
| `nonce`       | Text        | Base64 of the 96-bit GCM nonce, fresh per write.                               |
| `key_id`      | String      | The hub-key generation that encrypted this row.                                |
| `revision`    | Integer     | 1 at create, incremented by every replace.                                     |
| `replaced_at` | UtcDateTime | When the current value was written.                                            |
| `replaced_by` | String      | Who wrote it.                                                                  |
| `created_at`  | UtcDateTime |                                                                                |

The associated data authenticated with each ciphertext is `name` and `revision`, so a ciphertext copied onto another row
or replayed at an older revision fails to decrypt. `secret_lifecycle_facts` carries retirement in the
`scope_lifecycle_facts` shape. Retiring a secret that an active record references is refused.

Encryption uses `cryptography`'s AEAD primitives, already present through `pyjwt[crypto]`.

## Write-only

| Verb    | Route                             | Body            |
| ------- | --------------------------------- | --------------- |
| create  | `POST /api/secrets`               | `{name, value}` |
| replace | `PUT /api/secrets/{name}/value`   | `{value}`       |
| list    | `GET /api/secrets`                | —               |
| show    | `GET /api/secrets/{name}`         | —               |
| retire  | `POST /api/secrets/{name}/retire` | `{by}`          |
| enable  | `POST /api/secrets/{name}/enable` | `{by}`          |

A secret's view carries its name, revision, `replaced_at`, `replaced_by`, retirement, and the records that reference it.
No route, response, or CLI verb returns a value or a ciphertext. `blizzard hub secret set NAME` reads the value from
stdin, never from an argument, so it stays out of shell history and the process table; it creates the secret when the
name is new and replaces it otherwise. A replace appends a `config_changes` row with `op = replace` and no diff
([records.md](./records.md) §The change log).

## Reading a value

Decryption happens inside the hub process at the moment a value is used, through `ISecretReader.reveal(name)`, which
returns a `SecretValue` whose `repr` and `str` are redacted. Only objects built by the composition root receive an
`ISecretReader`: work-source adapters, forge clients, `GitHubCommitResolver`, and the hub step's `HubEnv`
(`hub/delivery/hub_node.py`). Controllers hold `ISecretCatalog`, which exposes metadata and references only
(`bzh:controller-read-only`).

A value is never written to a log line, a span attribute, `event_log`, `config_changes`, an apply outcome, or a
fact-egress row. The one place it leaves the process is `BZ_FORGE_TOKEN` in a `run:` step's environment, as
`bzh:hub-node-env-contract` already provides.

## The hub key

The signing-key lifecycle in `hub/auth/signing.py` is the precedent: key material never sits in the store or the config
file, its directory is owner-only (`0700`, files `0600`), and a `meta.json` names a `current` and a `previous`
generation so rotating never strands what the older key protected. The secret store's key follows it.

- **Where it comes from.** `BZ_HUB_SECRET_KEY` (base64, 32 bytes), when set, is the only generation. Otherwise
  `data/auth/secret-keys/`, laid out as `signing-keys/` is, holds the generations. `blizzard hub init` creates that
  directory and a first generation when the variable is unset. The key reaches the domain through an `IHubKeyProvider`
  seam (`bzh:pluggable-seams`); an external key service is a later binding behind it.
- **When it is missing.** A hub that holds any `secrets` row and finds no key generation matching a row's `key_id`
  refuses to start, naming the missing generation — the same shape as a store-revision mismatch under
  `bzh:manual-migrations`.
- **Rotation.** `blizzard hub secret rotate-key` mints a new generation, re-encrypts every row under it in one
  transaction, then demotes the old generation to `previous`. Reads use each row's `key_id`, so a hub reading
  mid-rotation decrypts both. Under `BZ_HUB_SECRET_KEY`, rotation takes the old key as `BZ_HUB_SECRET_KEY_PREVIOUS` for
  the duration.
- **Loss.** Without the key, values are unrecoverable. Records keep their references; every secret is set again.

## References

Work sources and repositories name a secret by `secret_name`. A write naming a missing or retired secret is refused with
422, so a dangling reference is caught when it is written, never when it is used. Nothing reads a credential from the
hub's process environment once `token_env` is gone; that absence is what lets a secret belong to a tenant.

## Toward tenancy

`epic:multi-tenancy` adds `tenant_id` to `secrets`, makes the key `(tenant_id, name)`, and adds the tenant id to each
ciphertext's associated data, so a row moved across tenants fails to decrypt. Names, routes, and the reveal seam keep
their shape.

## Replacing a value

A replace writes new ciphertext and a new revision, and the old value is gone. It takes effect on each referencing
record's next use ([runtime.md](./runtime.md)). An operator who needs an overlap keeps both tokens valid at the forge
while the replace propagates; the hub keeps one value per secret.
