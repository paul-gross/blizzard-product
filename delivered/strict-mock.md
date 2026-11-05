---
epic: strict-mock
refinement: pristine
status: delivered
---

# Plan — `epic:strict-mock`

`epic:strict-types` held blizzard to strict pyright, and left blizzard-mock as the one Python codebase in the fleet
whose checker still looks away. The mock is not a side project: every service and fleet test blizzard runs talks to it,
it is held to the same Python toolchain (`bzh:python-toolchain`), and agents read and copy its code exactly as they do
blizzard's. A signature there makes the same promise, and nothing keeps it.

Measured on `master` at `dc944081b`, strict mode with the blanket comments ignored reports about 1,700 errors — some 360
in `src` and 1,300 in the tests — behind 55 `# type: ignore` comments. Two thirds of the test half sits in two files,
the harness smoke tests and the mock hub tests, and most of the source half in the harness's parsed OpenCode documents.

## How it lands

The strategy is `epic:strict-types`'s, scaled down. A first landing deletes the comments that silence nothing and adds a
strict report that gates nothing, so every slice measures its group before it starts and proves it gone when it
finishes. Pyright stays in `standard` while the count comes down, with no baseline and no ratchet. The slices then pay
the debt down one reason and one area at a time, largest group in the smallest area first: the source packages before
the tests, since typed source clears many of the tests' knock-on errors, and the private-name reach last among the
tests, since it changes how a test is written rather than how it is typed. Every slice is behavior-preserving, and the
mock's wire stays byte-identical throughout.

## Turning it on

The last slice sweeps what no longer clusters and switches the gate: pyright strict over every Python file in the repo,
not a list of folders, the gate rejecting a `# type: ignore`, the strict report retired, and `bzh:strict-types` widened
in `blizzard-context` to cover the mock. With that, every Python file in the fleet is strict.

## Out of scope

`blizzard-infra`, whose typing is its own repo's concern, and any behavior change to the mock beyond what a real type
bug requires.
