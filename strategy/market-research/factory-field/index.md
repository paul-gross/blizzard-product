# The software-factory field

Blizzard is not the only fleet shipping code while its owner sleeps. On public GitHub there are now a few hundred
repositories with one person behind them, landing pull requests faster than a human could read them. A handful are
products that call themselves factories; most are ordinary codebases (an input method, a chess app, a Vue toolchain, a
trucking system) whose owner quietly put a fleet to work. This folder sizes that field. Fusion, the nearest single
competitor, keeps its own folder at [fusion-ai](../fusion-ai/index.md); this one is about the crowd around it.

| File                               | When to read                                                                                                                  |
| ---------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| [landscape.md](./landscape.md)     | Looking up a specific factory: its throughput, its human count, which bucket it landed in, and the evidence that put it there |
| [landscape.html](./landscape.html) | Seeing the field at a glance — guardrails plotted against health, the bucket mix by throughput, and the same tables to browse |

## What the field says

Speed is not what sets anyone apart. Dozens of solo repositories merge more than fifty pull requests a day, and the
busiest pass 150. Blizzard's thirty-day average is about twelve a day, rising to twenty-five in its most recent week.
Fusion, which looked fast in the spring, has dropped below one a day. A fleet that merges quickly is now easy to build.
A fleet whose codebase is still in good shape after three months at that pace is not.

Structure is rare, and it gets rarer as volume rises. Of the 113 factory-built codebases measured, 27 qualified as
well-structured autonomous systems, 48 had real guardrails under visible strain, and 38 had too few guardrails to call
the output engineered. Among the 37 repositories merging fifty or more pull requests a day, only seven were
well-structured. The median well-structured repository merges about 23 pull requests a day and keeps its default branch
green 88% of the time. The median strained one merges about 34 and is green 67% of the time.

Strain looks much the same everywhere. The default branch goes red after merge and stays red. Single source files grow
into megabytes: one orchestration harness has a 2.9 MB `run-task.ts`, and Fusion's self-healing module is 17,900 lines.
Agents commit their own verification logs into the tree by the hundred. Work bypasses the pull request and lands
directly on `main`. A rising share of commits repairs the system rather than extending it. None of these is a mystery;
each is what a fleet does when nothing in the loop says no.

A surprising share of the volume is not software at all. Twelve of the 67 busiest repositories turned out to be formal
mathematics, research ledgers, crawled data, generated content sites, or governance records. They have a fleet's cadence
but not a codebase, and they are set apart so they do not flatter or damn the comparison.

## What it asks of blizzard

The position worth holding is structure at volume. Blizzard's gates, its enforced layering, its crash sweep, and a
default branch that is mostly green are what put it in the top bucket. They are also the reason its pace is moderate
rather than extreme. The open question is whether blizzard can roughly double its throughput without leaving that
bucket. Few in the field have managed it; [Nishfleet/0509](https://github.com/Nishfleet/0509),
[JovieInc/Jovie](https://github.com/JovieInc/Jovie),
[prime-radiant-inc/evener](https://github.com/prime-radiant-inc/evener), and
[sjawhar/legion](https://github.com/sjawhar/legion) are the peers to read for how.

Blizzard's own weak spot is one of the field's commonest symptoms. About one merge in five turns `master` red even
though its pull request passed. That is the strained bucket's signature on a small scale, and the merge-skew problem the
busier factories never solved.

The factories that build themselves are the closest analogues to blizzard and the most useful to watch:
[GoogleCloudPlatform/scion](https://github.com/GoogleCloudPlatform/scion),
[craigoley/remudero](https://github.com/craigoley/remudero), [rjwalters/loom](https://github.com/rjwalters/loom),
[vtmocanu/uzi](https://github.com/vtmocanu/uzi), [orbi-build/orbi](https://github.com/orbi-build/orbi), and
[jeong-sik/masc](https://github.com/jeong-sik/masc). Every one of them merges faster than blizzard, and every one landed
in the strained bucket.

## How the field was found

Three searches ran independently, and each repository was then measured the same way. The first combed
[varun1505/awesome-software-factories](https://github.com/varun1505/awesome-software-factories), GitHub topics, and
named factory tools, then followed each tool to the repositories that use it by the marker it leaves behind: a `.beads/`
directory, a `Fusion-Task-Id:` trailer, `polecat/` branches, or a factory bot account. The second streamed sixteen days
of [GH Archive](https://www.gharchive.org/) and ranked repositories by push activity among those with three or fewer
human pushers. The archive records only a few percent of merge events, so its counts served to nominate candidates and
never to measure them. The third searched pull requests opened by coding-agent bots, and `claude/`, `codex/`, and
`cursor/` branches, in early October.

Every candidate was then measured over the thirty days to 2026-10-08. Merged pull requests per day and commits per day
come from GitHub's search and history APIs. Lines added per day come from the default branch's last seven days where
that was cheap to fetch; otherwise they are marked ≈, estimated from the average size of the last hundred commits times
the commit rate. Humans counts contributors with twenty or more commits after removing bots and agent accounts. A number
in parentheses is the minor humans, each with under a tenth of the lead's commits.

## How the buckets were drawn

Each codebase was read from a shallow clone of its current tree and its recent commit subjects, and scored on two axes.
Repository code was never run.

**Guardrails** asks whether the repository is built to refuse bad work. It rewards a pull-request gate that runs tests,
lint, and a type check; boundaries enforced by a tool (dependency-cruiser, eslint boundaries, import-linter, ast-grep,
cargo-deny or denied clippy warnings, knip or madge); strict typing; a test suite at least comparable in size to the
source; and disciplined commit messages. **Health** asks whether the refusals are working. It rewards a default branch
that is mostly green after merge and penalises oversized source files, rework-heavy history, checkpoint commits,
committed logs and backups, sprawling root-level markdown, and work that lands without a pull request.

- **Well-structured autonomous system:** strong guardrails, and health above zero.
- **Structured but strained:** strong guardrails with poor health, or partial guardrails with good health. Fusion lands
  here: its ratchets and gates are real, but its main branch is green 3% of the time.
- **Vibe-coded mess:** few guardrails, whatever the outcomes.
- **Not software:** set aside by inspection before scoring.

The scores are a screen, not a verdict. They read configuration and history, not design, and they miss what a repository
does outside GitHub Actions. A borderline repository can move a bucket on one config file. Read the evidence column
before quoting a placement.

Findings are recorded as of 2026-10-08.
