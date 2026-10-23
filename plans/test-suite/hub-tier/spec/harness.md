# What the harness gains

The foundation's harness carries the three analytics ports. The hub's other tests need people, the CLI, a configured
fresh hub, and somewhere to export traces, and each arrives in `tests/harness/` as a fixture under
[the foundation's rules](../../foundation/spec/package.md#files).

## Profile placeholders

A profile is static TOML, but the `auth` profile must name its hub's own IdP as its provider, and the IdP's port is
known only once it starts. A profile's string values may therefore name the hub's mock coordinates: `${idp}` and
`${forge}` expand to the stub IdP's and the mock forge's base URLs when the hub's file is written. Nothing else expands,
and an unknown `${…}` fails the profile check before any hub starts. The key check is unaffected, since placeholders sit
only in values.

## People

The `auth` profile turns authentication on with the stub IdP as its one provider:

```toml
[auth]
mode = "oauth"
superuser = "superuser@blizzard.test"

[[auth.oauth.provider]]
name = "stub"
type = "oidc"
display_name = "Stub"
issuer = "${idp}"
client_id = "blizzard-tests"
client_secret_env = "BLIZZARD_TEST_IDP_SECRET"
emails_authoritative = true
```

```toml
[auth.login_throttle]
enabled = false
```

The harness sets `BLIZZARD_TEST_IDP_SECRET` in that hub's environment.

The login throttle gives each client address ten tokens, refilled one every six seconds, and a sign-in spends two, one
at `authorize` and one at `callback`. Every test on a shared hub signs in from `127.0.0.1`, so a handful of sign-ins
would exhaust it for the whole hub. The `auth` profile turns it off through the keys
[surfaces.md](./surfaces.md#a-configurable-login-throttle) adds. A test whose subject is the throttle takes `fresh-hub`,
keeps the throttle on, and may set its capacity through `hubConfig` so its case is reached in a few requests.

Signing a person in is the same walk wherever it happens: the harness registers the person's identity with the IdP
(`POST /_levers/identities`), starts at the hub's `GET /api/auth/stub/authorize`, follows the hub's redirect to the IdP
with `login_hint=<email>` added, follows the IdP's redirect to `callback`, and keeps the request context holding that
person's `bz_session` cookie. The walk is the real flow with no browser: the IdP's `authorize` shows no page and
redirects straight back with a code.

Two fixtures make people, and a test names whichever it needs:

| Fixture                   | What it is                                                                                                                                                                                                             |
| ------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `user`                    | One signed-in person who belongs to the test, in `tenant`, with the role the `userRole` option names (default `contributor`). `{ email, role, context, api }`, where `api` is a client acting in that person's session |
| `people.create({ role })` | A factory for every further person: the inviter and the invitee, a guest beside an admin. Each call returns what `user` is                                                                                             |

`userRole` is an option, set with `test.use` beside `isolation`, so a file of contributor tests declares its user once
and no test body makes one. Both are built the same way: an invitation through `POST /api/invitations` as the tenant's
admin, with an email built from the test's id and a random suffix, then the sign-in walk with the invitation's token on
`authorize`'s `invite` parameter. The invitation and the person are recorded in the ledger as the test's own, under a
`person` entry in `resources.ts`. A person only the test knows is invisible to every other test, so inviting, signing
in, changing that person's role, and removing them are fine at `common`; the same change aimed at the tenant's admin, or
at a tenant-wide setting such as which providers may sign in, is not, and needs `fresh-tenant`.

`user` and `people` need a hub with authentication on. On a hub with it off there is nobody to sign in, and a test
naming either fails before it starts, naming the `auth` profile.

On an `auth` hub, the setup project signs the superuser in once, which claims the bootstrap, and creates the common
tenant as that person, and the harness's internal client acts in the superuser's session. A `fresh-tenant` tenant is
created the same way, with the superuser as its admin, and `api` acts in the superuser's session, as the tenant's admin.
Acting as anyone else is `user.api` or a person's own `api`, always explicit. A test whose subject is the bootstrap
itself takes `fresh-hub`, where nobody has signed in yet.

## The CLI

`cli(args, { as? })`, a factory fixture, runs the installed `blizzard` binary, from the same `.venv/bin/` as the
daemons, with the contract's environment, `BZ_HUB_URL` set to the test's hub, `BZ_HUB_TENANT` to `tenant`, and
`XDG_CONFIG_HOME` to an empty directory of the test's own, so no session but one the test made is ever read. It returns
`{ code, stdout, stderr }` and fails the test on a timeout of thirty seconds. With `as: user` or any person `people`
made, the CLI is first signed in as that person through `blizzard hub login`, walked through the IdP as above.

A verb the CLI runs keeps to the test's level by the same rules as a call through `api`.

## A fresh hub's own configuration

A test at `fresh-hub` may configure its hub beyond its folder's profile, since nothing else shares it:
`test.use({ hubConfig: { … } })` merges keys over the profile, and `hubEnv: { … }` adds variables to the hub's
environment, which cannot remove a contract `set` variable. Both are checked as a profile is, and both are refused at
every other level, so a hub-wide setting cannot reach a shared hub through a test.

A fresh hub also hands its test its lifecycle, through the `freshHub` fixture, refused below `fresh-hub`: `stop()` sends
`SIGTERM` and resolves when the process exits, with its exit code; `kill()` sends `SIGKILL`; `start()` hosts the same
directory again on the same port. The SIGTERM test and the tracing tests' flush use them.

## The OTLP sink

`harness/otlp-sink.ts` is an OTLP/HTTP receiver on a port of its own, accepting what `tests/otlp_sink.py` accepts today
and recording each export's spans with their attributes. `otlpSink()`, a factory fixture, starts one for the test and
returns its endpoint, which a `fresh-hub` test names in its `hubConfig` tracing settings, and its spans, which the test
reads once the hub has stopped and flushed.
