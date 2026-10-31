# Surface behind a flag

A flag hides behavior, never contract (`bzh:flag-hides-behavior`, a new rule under `blizzard-context`'s `standards/`).
Surface a change ships is released when a release carries it, whether or not its flag is on, and is held to
`docs/versioning.md` from that release on. Turning a flag on changes what the software does with that surface; it never
adds, removes, or reshapes the surface itself, and turning one off again — an override set back to `false` — never takes
any away.

## Each surface while its flag is off

| Surface                      | While the flag is off                                                                                                                                                                                              |
| ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| HTTP route, hub or runner    | Registered and in the OpenAPI document. Answers `404` with a typed `FlagOff` body naming the flag, declared once in `wire/` and listed in the route's `responses`, so a client tells *not on yet* from *not found* |
| Request or response field    | Declared optional on the model. A request carrying it while off is accepted and the field ignored; a response carries its default — `None`, an empty list or map                                                   |
| Hub↔runner wire              | As a route or field above, held additive by `bzh:fleet-wire-additive`. A runner never sends a field or calls a route its `fleet` flag has off                                                                      |
| CLI verb or option           | Present in `--help`, marked as not yet on. Running it while off exits non-zero with the same message the route's `FlagOff` carries                                                                                 |
| Configuration key            | Accepted and validated like any key, so a mistake is caught while off. Ignored while off, with a warning naming the flag                                                                                           |
| Store schema                 | Migrated by every release that carries the revision, regardless of any flag. A table or column only the off path writes stays empty. A migration is never conditional on a flag                                    |
| Trace span, event, attribute | Declared in `contracts/traces/dictionary.json` when the change that emits it lands; emitted only while on                                                                                                          |
| Egress column                | Declared in `contracts/egress/dictionary.json` as a nullable column when it lands; null while off                                                                                                                  |
| Fact kind, event type        | Declared when it lands; written only while on. A reader that meets one, whatever its own flag, reads it                                                                                                            |
| Board screen or control      | Absent. The board is behavior, not surface, so it is what a `hub` flag most often hides                                                                                                                            |

## The checks never read a flag

Every check of `epic:release-engineering/compat-floor` compares the surface a release carries — routes, fields,
configuration keys, schema, the trace and egress dictionaries — and none of them evaluates a flag. That is what keeps
them sound: a check that skipped surface behind an off flag would let it be reshaped after it had already shipped, and
an operator holding a client written against the off route would find it broken the morning the flag turned on.

What follows for the work is a discipline, not a check. An epic whose surface is still moving keeps that surface off
`master` until it settles, or ships only surface it is content to support for a major; the flag buys time for behavior,
never for shape. Removing surface a flag guarded, if the work is abandoned, is a removal like any other, carried by a
major's cycle under `epic:release-engineering/compat-floor`.

## The mock

`blizzard-mock` mirrors every route and field a flag guards when the change that adds it lands, as it mirrors any other
(`bzh:wire-change-extends-mock`), and answers it as the flag's on path does. It reads no registry of its own, since it
never imports `blizzard`. `mock_hub`'s `/api/fleet/identity` answers `fleet_flags` empty unless a lever sets names in
it, so a service test drives the real runner through a fleet flag's on path and its off path against the same mock;
`mock_runner` drives the real hub with whatever the hub under test resolves.
