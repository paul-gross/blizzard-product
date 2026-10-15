# You Are the Choke Point: Three Days, Two Epics, and a Thousand Disposable Agents

*Written in the first person by the orchestrating Claude session (Opus 5.5) that built and shipped blizzard's
multi-tenancy and projects epics.*

At 04:39 CDT on October 8, Paul handed me the largest single change blizzard has taken on: "I want you to build out an
entire epic for me. We've done this once before with the architectural sweep… We're going to use dynamic workflows, so I
want you to orchestrate and manage a fleet of agents. Subdivide and conquer. Don't use a winter workflow, use your own."

The epic was really two, built as one:

- **Multi-tenancy** makes the tenant the one boundary inside a hub. One installation hosts several wholly separate
  worlds, each with its own graphs, chunks, runners, findings and board, and none can see or reach another.
- **Projects** gives a tenant first-class groupings of work. Each project owns its scopes, routines and garden, links
  the sources it draws from and the repositories it lands in, and carries every chunk from ingest to landing.

The brief came with four feature environments, alpha through delta, and "full exclusivity" over them. It also held one
sentence that turned out to describe the job better than anything else in it: "You are the choke point. Take as long as
you need, there is no rush." A minute later came the only hard limit: "Manage the context of your agents, keep them
under 400k please."

Seventy-three hours later, three of them paused because the week's usage ran out, the hosted hub was serving the result
from its real store, migrated in place:

- 274,649 rows were carried into a `default` tenant.
- Every work source and repository was linked to a `default` project.
- Both production runners were rebuilt, migrated and serving it.

Paul's part was decisions, not code review. Paul answered eleven product questions, approved one merge strategy, and
wrote the deploy brief.

| What                | How much                                                                              |
| ------------------- | ------------------------------------------------------------------------------------- |
| Wall clock          | about 73 hours, Oct 8 04:39 → Oct 11 05:50 CDT, including a three-hour usage pause    |
| Workflow runs       | 68 (63 completed, 5 stopped), from 60 scripts totalling 10,771 lines                  |
| Agents              | 958 by the runs' own count, plus 7 run on their own                                   |
| Tool calls / tokens | about 22,800 / about 76.5 million, by the runs' own accounting                        |
| My own context      | compacted 14 times, each time from between 395k and 684k tokens                       |
| Delivered           | 212 commits: blizzard 182 (+82k / −10k lines), blizzard-mock 11, blizzard-context 19  |
| Schema              | 15 new migrations, 13 in the hub and 2 in the runner                                  |
| Tests               | full suite from 12,117 to 13,527 passed, 0 failed                                     |
| Paper trail         | 19 issues, 3 PRs, every commit signed, every child issue closed by exactly one commit |

A *dynamic workflow* here is a script I write and run: it fans work out to many agents with deterministic control flow,
structured results and phases. One Claude Code session — me — wrote those scripts, launched them, read their results,
and kept the state of the whole build.

This post is about how that worked: the process, the failures that shaped it, and — because Paul asked for exactly this
— which parts came from blizzard and winter, which came from my training, and which I had to build on the spot. It ends
with what the build says blizzard itself needs, because a change like this one should next run inside the fleet, not
beside it.

---

## The shape of three days

| When (CDT)           | What happened                                                                                                          |
| -------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| Oct 8 04:39          | The mandate; four environments provisioned by 04:40                                                                    |
| 04:46                | **Survey:** 18 read-only surveyors, 2 checklist builders, 2 synthesizers                                               |
| 05:20 → 05:37        | Ten decisions sent to Paul; answered in two messages                                                                   |
| 05:35                | **P0**, ground: 12 child issues, frozen migration revisions, a seeded pre-epic store, verification tools               |
| 06:00 → 21:15        | **P1–P4**, the tenant key bottom up through the store: seam and guard, scoped adapters, key switched on, keys rebuilt  |
| 21:14 → Oct 9 03:50  | **P5**: services split, identity carried over, every request resolves its tenant                                       |
| Oct 9 03:52 → 17:10  | **P6**, tenant surfaces: administration, invitations, deletion, per-tenant sweeps, the CLI — across three environments |
| ~06:00               | Five concurrent workflows push the load to 37; the operating system kills the terminal                                 |
| 12:18 → 12:28        | An eleventh decision, raised by a review finding: "Yes it should absolutely!"                                          |
| 14:29 → 21:21        | **P7**, the multi-tenancy checkpoint: assembly, every tier, skew rehearsal, cold evals, a board pass                   |
| 17:11 → Oct 10 02:41 | **P8**: the projects model and its links                                                                               |
| Oct 10 02:41 → 21:37 | **P9**: projects through ingest, delivery, eligibility, the runner and the board                                       |
| 08:05 → 11:01        | Usage pause; three continuation scripts on resume                                                                      |
| 21:40 → 23:09        | **P10**: assembly, history normalization, the final gate, three cold-eval fix rounds; three PRs opened                 |
| 23:25 → 23:30        | CI green, 21 of 21 checks; the local full suite at the PR tree: 13,527 passed                                          |
| Oct 11 04:47         | "Tell me how you code reviewed and verified all of it please!"                                                         |
| 05:05                | Paul hands me the deploy                                                                                               |
| 05:14 → 05:21        | Backups, a rehearsal on real data, runners stopped, three merges                                                       |
| 05:43 → 05:50        | The hosted hub migrates its real store in about four and a half minutes; runners redeployed; done                      |

By the labels they carried, about 300 agents verified something, about 230 built, about 200 committed, 83 ran or judged
cold evals, and 50 reviewed. Fewer than a quarter of the agents wrote code. That ratio is the build in one number:
construction was the cheap part.

---

## A playbook written for people, read by an agent

Paul's first message pointed at a file: the retrospective the previous orchestrator wrote after the architectural sweep,
[nine-sweeps-one-line.md](./nine-sweeps-one-line.md), which sits beside this one. "That contains a lot of great info on
how we accomplished our goal." I read it before planning anything, and my draft plan cites it by line. It already
described most of the machinery this build ran on:

- commits split into a scripted ("mechanical") half and a hand-written ("judgment") half, with the scripted half
  replayed byte for byte;
- reviewers paired with skeptics whose job is to refute them;
- cold-spawn evals for every new rule in the harness;
- agent budgets counted in tool calls, a context watchdog, and a CPU semaphore for a machine shared with the live fleet;
- the resume-cache trap, the drifting-working-directory trap, and a ledger on disk that outlives the context window.

So much of what follows was inherited and hardened rather than invented, and I want to say that plainly. It is the most
useful thing I learned: a retrospective written for humans worked as operating instructions for the next agent. The
previous run's "what I would do differently" list became my defaults on day one: budget agents from the start, name one
owner per shared file, watch master. A post like this one is part of the harness.

The inheritance had limits. The previous run lasted about eighteen hours; this one lasted seventy-three. Most of what I
had to add — a memory cap, a `/tmp` sweep, a rule about resuming runs, a history normalizer — came from failures that
only appear when a build outlives a dozen context windows, a crash and a usage pause.

---

## Survey first, then ten questions

Before any code, a survey workflow put 18 read-only agents over the specs, the code and the harness. Two checklist
builders and two synthesizers then turned their reports into a plan. The output:

- two checklists of acceptance items, 346 for multi-tenancy and 318 for projects;
- an 82-lane draft plan;
- a decision file.

The plan synthesizer itself ran to 383k tokens of context and had to be stopped. It was the same failure the previous
run reported: a single agent asked to hold everything. Synthesizers since then get small inputs and are forbidden to
open full reports.

The decision file sorted every open question by who owned it:

- **S1–S20** were already settled by the documents. The spec said it, so nobody needed asking.
- **E1–E61** were engineering calls. I made them and recorded the reason for each.
- **PD-1–PD-38** were plan decisions about how to build.
- **D1–D10** were Paul's: product shape and risk appetite.

Ten questions out of roughly 130 went to a person, and Paul answered them in two messages twelve minutes apart. The
answers were not rubber stamps. Paul took my recommendation on six, rejected it on two, and rewrote the question on two
more:

- **D1, where the tenant lives.** I proposed resolving a tab's tenant per request and keeping the board's eighteen
  permission checks as they were. Paul's first reply was "I don't understand. Explain more." The second reframed the
  board: "the user selects a tenant at login and throughout the entire session they have that tenant… nothing renders
  without a selected tenant… I was expecting the tenant concept to be solved and /api/me to be solved at a higher level
  closer to the shell rather than in each individual tab." That paragraph moved the tenant above the shell for every
  view.
- **D2, a call that needs a project but names none.** I proposed resolving a name across projects and answering 409 on a
  collision. Paul: "It should use the default project if there is only one … or it should fail." No guessing: one
  project, use it; several, `422 project_required`.
- **D7, the CLI's defaults.** I proposed a `blizzard hub context` command family. "I don't want some sort of 'context' I
  dont understand what this is — the CLI should use a default tenant/project if there is only one… but otherwise it
  should require it on each execution." A whole subsystem disappeared.
- **D9, two tenants annotating one GitHub repository.** "I don't understand please explain…" and then: "It's a luxury
  feature." Build nothing.
- **D4, an ingest form on the board.** "No."

One more question surfaced mid-build. A review of the identity work found that an address from a sign-in provider that
does not own its users' email was trusted in two places: to claim the configured superuser, and to merge a new sign-in
into an existing account. The skeptic refuted it as a defect of the change, since the gap predated the epic, but judged
it real. So it went to Paul as a product question. "What does that mean what are you asking?" Then, once explained: "Yes
it should absolutely! Thanks for raising this." It shipped as D11 that afternoon. It is the clearest case in the build
of review earning its keep: not a bug in the work, but a hole the work made visible.

Two patterns are worth recording for blizzard:

- **The overrides all pointed one way.** Paul chose stricter, simpler and fewer: refuse rather than guess, flags rather
  than saved state, nothing rather than a luxury.
- **Three of the eleven questions needed a second explanation.** I wrote them the way an engineer frames them, in
  mechanisms, endpoints and error codes. Paul answers in terms of what a person sees and does. The questions that landed
  the first time began with the user's experience.

---

## Eleven phases, four environments, one committer

The plan ran in eleven phases, P0 to P10, each closed by a gate:

- **P0** laid ground. It filed the twelve child issues and froze the old migration revisions so new ones could not
  import live tables. It also built a seeded pre-epic store with a row in every table, and the verification tools.
- **P1–P5** drove the tenant key up through the hub store, bottom first:
  - a scoped seam and a statement guard;
  - every adapter statement scoped, through one codemod and eight judgment lanes;
  - the key switched on, with its backfill;
  - keys and columns rebuilt;
  - then services, identity and the event log.
- **P6** built the tenant surfaces — administration, invitations, deletion, per-tenant sweeps and exports, the CLI's
  `--tenant` — across three environments at once.
- **P7** was the multi-tenancy checkpoint. It assembled everything, ran every tier, rehearsed version skew, evaluated
  the harness changes cold and walked the board.
- **P8 and P9** did the same for projects. P8 built the model and its links. P9 carried projects through ingest,
  delivery, eligibility, the runner and the board.
- **P10** assembled, normalized and shipped.

The environments had fixed roles:

- **Alpha** held the backend line. It was the critical path, touched by one workflow's committer at a time.
- **Beta** held the web app, the docs and the blizzard-context line.
- **Gamma** held the runner and the mock.
- **Delta** did nothing but verification. Every use of it ran under one file lock, because two gates sharing an
  environment's ports produce failures that belong to neither.

Inside a workflow, agents had narrow roles:

| Role                            | What it did                                                                                               |
| ------------------------------- | --------------------------------------------------------------------------------------------------------- |
| Lane                            | built one planned commit or a few, in its own worktree, against named spec sections                       |
| Committer                       | the only agent allowed to commit, reset or rebase; serialized, so one at a time                           |
| Replayer                        | checked each landed commit alone at its parent: build, lint, types, structural gate, the tests it touches |
| Gate                            | ran a phase's verification tiers at its end                                                               |
| Reviewer and skeptic            | one found problems along a dimension; one or two others tried to refute each finding                      |
| Cold-eval designer, runs, judge | tested new harness rules on agents that had never seen them                                               |
| Browser verifier                | walked the board in a real browser against the environment's own hub                                      |
| Fixer, rebaser, planner         | the rest                                                                                                  |

A few patterns did most of the work:

- **Relay lanes.** An agent cannot see its own token count, so its budget is tool calls: hand off at the budget minus
  fifteen, stop at the budget. At a handoff it writes a note. The next agent starts by verifying the note against the
  files instead of redoing the work. The shared preamble says it outright: "Handing off is success, not failure; running
  past the budget is the failure."
- **One committer, many builders.** Lanes return their commits as data, a message and an exact file list. A single
  serialized committer stages exactly that list, and nothing else may commit, reset or rebase.
- **Every commit green alone.** A second queue replays each landed commit at its parent. A codemod commit carries its
  script and must replay byte for byte: "fix the script, never its output."
- **A commit graph instead of waves.** Through P8 the lanes ran in waves. P9 had too many cross-dependencies for that,
  so each lane became a promise that resolves to its landed commit and awaits exactly the commits it stands on.
- **Continuations instead of resumes.** Why is in "What went wrong" below.

Every prompt opened with a shared preamble:

- It quoted Paul's authorization verbatim.
- It listed the hard safety rules. Never point at the hosted hub. Never kill a process by pattern. Never touch the
  source checkouts. Use `git -C` everywhere. Only the named committer commits. Stay inside your files. Put no process
  references in anything you write.
- It told the agent how to behave on a machine shared with the live fleet.

Every judgment call went into an audit file per lane, each row naming the decision it applied. A product question
returned the lane as `blocked` rather than guessed.

---

## The verification ladder

Paul's first question on the last morning was "Tell me how you code reviewed and verified all of it." The honest answer
is: in layers. Each layer left evidence, and none of them had to be perfect.

- **Per commit.** Every one of the 182 blizzard commits builds, lints, type-checks and passes its impacted tests on its
  own. Every codemod commit replays byte for byte.
- **Per phase.** Gates ran the full suite and a checkout matching CI's Python 3.12. Each phase's adversarial review ran
  before the next phase built on it. Across P1–P6, about 65 findings were raised and about 43 survived refutation and
  were fixed. The first review alone confirmed 13 of 14, including a race in how SQLite begins deferred transactions. A
  later one found the tenant backfill was quadratic, and the fix took a million-row backfill from 29.6 seconds to 1.4.
- **Upgrade safety:**
  - A seeded pre-epic store, with a row in every table, was migrated and all 163 of its read routes smoke-tested.
  - A comparer migrated the same fixture with the before and after code and diffed the API's answers.
  - A million-event store migrated in 18 seconds, with a disk peak of 3.8 times the store. Those numbers went into the
    deploy notes.
  - A skew rehearsal ran an old runner against the new hub.
- **The tiers CI runs only at release, run locally:**
  - service (215), end-to-end (63), crash sweep (43), journey and the event-stream contract;
  - wire compatibility and OpenAPI drift;
  - 2,859 web tests plus bundle and generated-client drift;
  - the mock's 1,141 tests run against this blizzard;
  - the harness repo's ten declared verification methods.
- **What tests do not see.** Design-fidelity renders at 1280 and 390 pixels were compared against the product's HTML
  mockups, and one gate sent two board pages back to match them. A browser agent walked the board against a real hub.
- **The harness, cold.** Every new rule was tested on agents that had never seen it (next section).
- **At the end:** CI passed 21 of 21 checks across the three PRs, and the full suite passed 13,527 tests with 0 failed.

And then, at deploy time, came the one check nothing before it could provide: the migration rehearsed on a copy of the
real production store.

---

## The harness changed too, so it was tested like code

The build did not only read blizzard-context; it wrote to it. Nineteen commits added domain documents for tenancy and
projects and new rules: `bzh:tenant-scoped-store`, `bzh:hub-scoped-reads-confined` and `bzh:keyed-table-migration` among
them.

In this workspace a rule nobody can find does nothing, so canon's cold eval tests rules the way tests test code:

1. A fresh agent gets a realistic task that never names the rule.
2. Three such agents run each scenario.
3. A judge scores two things separately for each run: did it *reach* the rule, and did it *behave* as the rule asks.

At the checkpoint, three scenarios failed, at 0/3, 0/3 and 1/3. One failure was the eval's own fault: its expected
answer was wrong. The others were routing gaps, and after one harness fix all three passed 3/3.

At the final gate, three of five failed. The rounds fixed what the judges named:

- routing rows that did not catch the task as an agent would phrase it;
- rules written only as prose, which became *Detect* bullets an agent can check;
- one rule that did not say what it leaves unverified.

Three rounds and seven commits later, all five passed.

One wrinkle the method does not anticipate: cold agents read the *installed* harness, which is master, which does not
yet contain the rule under test. Every cold prompt had to send them to the environment's own checkout instead: "It is
newer than the installed copy … do not read that copy."

---

## What went wrong

A seventy-three-hour run finds failures an eighteen-hour run does not. In roughly the order they happened:

- **I crashed the machine.** At 06:00 on the second day I had five workflows running at once. The load reached 37 on a
  twenty-core box, and the operating system killed the terminal, and me with it. Paul's message on restart: the OS
  "killed wezterm which killed you due to CPU constraints." From then on:
  - at most three building workflows;
  - a job queue with three slots of six pinned cores, strict first-in-first-out, and web builds one at a time;
  - every heavy job under a 10 GB memory cap;
  - a watchdog that warns on load.
- **A test rewrote a real repository's git config.** A rebase that ran the runner's self-test on every commit exported
  `GIT_DIR`, and the self-test's `git -C <scratch>` calls inherited it. It committed its scratch baseline into gamma and
  rewrote the source checkout's config, with a fake identity and `core.bare = true`. Every commit for twenty minutes
  took the fake author. After the repair the rules became:
  - no more `rebase --exec`;
  - anything run under a rebase scrubs the git environment first;
  - the watchdog warns if a source repository's identity drifts.
- **Resuming a run replayed the wrong work.** The workflow tool resumes by replaying cached agent calls as an ordered
  prefix. After the crash, a resumed run's committer ran early, staged another run's unfinished edits, and committed
  them onto alpha under the wrong message. The rule since: resume only if every completed call precedes the first
  concurrent fan-out; otherwise write a continuation script. Every stop after that — overruns and the pause — ended in a
  continuation.
- **Builds ate the machine.** `ng test` reached 15 GB and `ng build` 37 GB, with 2 GB free; a lane had checked a commit
  in a copied directory with no git and no contracts. Later, esbuild reached 24 GB for a stranger reason. `/tmp` held
  1,379,560 directories leaked by a test helper since September, and esbuild lists every parent directory of what it
  resolves. A one-off sweep took `/tmp` from 1.4 million entries to 56 thousand, and the build ran in fifteen seconds.
  The leak is issue #1068, and a `/tmp` sweep joined the watchdog.
- **I lost track of where I was standing.** A bare `cd` at the moment of a compaction moved my working directory into
  alpha for three hours, and agents spawned in that window inherit it. Since then: `git -C`, absolute paths, subshells,
  and spawn only from the workspace root. It happened once more on the last morning and was caught in seconds.
- **I guessed times.** Writing timestamps from memory cost nine corrections in a day. The rule became: stamp a time from
  the clock inside the edit, never type it.
- **A blocked lane silently skipped six others.** In a commit-graph run, a lane that comes back blocked resolves to
  nothing, and everything awaiting it skips without an error. One missing edge between two issues would have quietly
  dropped six lanes. An ordering problem now waits and re-hops, and the edge list is part of what a plan must get right.
- **Gates passed by skipping.** Some tiers skip without the right environment variables, and a crash sweep without them
  passes vacuously. Worse, one gate ran a single service file instead of the tier and let 36 failures through to the
  checkpoint. The rule since: a track's gate runs the whole tier and every end-to-end file that drives its daemon, and a
  skip is not a pass.
- **The plan missed things.** The projects gate found seven service-tier proofs no lane had been assigned, and one
  checklist item that had never been built. Two gap-filling runs closed them.
- **Agents ignored budgets.** Two web lanes ran to 286k and 264k tokens of context without handing off. The watchdog is
  the backstop, not the prompt.
- **The week ran out.** At 08:04 on the third day: "Pause please our weekly usage is done." Everything stopped. Three
  hours later, three continuation scripts picked up where the stopped runs had been.

---

## Landing 182 commits

A line built by a thousand agents over three days does not arrive clean. Before it went up for review it was normalized:

- every commit re-signed, since the git plumbing used to squash commits does not sign;
- the self-test's fake committers replaced;
- process references stripped from code comments;
- every child issue closed by exactly one commit, its last.

A tool rebuilt the whole line from a plan and checked that every tree still matched. It ran three more times after the
pre-review checks found:

- a few comments the process-reference lint rejects;
- an intermediate commit over the line-length limit;
- a `Closes` on the wrong commit.

The three repositories had to land in order: mock first, blizzard second, and context immediately after, because
blizzard's tests read context's list of operations and fail when the two disagree.

Then GitHub refused to rebase-merge the 182-commit PR. It reported `rebaseable=false`, which Paul saw as merge
conflicts, although master had not moved and both of git's rebase engines replayed the line locally to an identical
tree. "Github says our PR has merge conflicts? how is that possible?" It was not possible: GitHub's server-side check
gives up on some large PRs without saying why. The contributing guide allows a merge commit when grouping a branch tells
a clearer story, and two epics in 182 commits qualify. A merge commit also kept every commit's signature, which a GitHub
rebase would have stripped. "Merge commit sounds great! Make it so."

---

## The deploy: Paul's brief, my checklist

The deploy brief was Paul's, and it was a runbook in five sentences: "First, we're going to want to ssh into the hub and
backup our db and image just in case. I don't want to lose the DB. Next, we'll spin down our runners and merge the PR…
Once it is deployed on the hub we should set up the default env and make sure it's all working w/ our setup. From there,
we configure our runners locally and bring those up."

- **Backups, four ways.** The host has no SSH, so everything ran over AWS Systems Manager. The hub was stopped for 18
  seconds for an application-consistent copy of its 667 MB SQLite store and the keys that seal its secrets, and the
  running image was saved too. Copies went to the host, to S3, to this machine and into an EBS snapshot, each checked by
  hash.
- **A rehearsal on real data.** The local copy let me do what no fixture could: run the new migrations against the real
  store before merging anything.
  - The migrations took 6.8 seconds.
  - 90 of the 91 existing tables kept identical row counts; the one that changed gained the nine configuration records
    the projects migration writes.
  - 274,649 rows landed in `default`, and Paul came out as its admin and as hub administrator.
  - The integrity check found four foreign-key violations, and the backup turned out to hold the same four, from
    transcript segments whose chunks were deleted long ago.
- **The wire gate, run before the merge.** Master's image publish is gated on a wire-compatibility check against what is
  deployed. I ran it locally first: additive.
- **The update, by hand.** The hosted hub updates itself from the `edge` image on a five-minute timer, and fences any
  release that is not ready within 120 seconds. I paused the timer and ran the updater myself with a ten-minute window.
  The migration that took 6.8 seconds here took four and a half minutes on the hub's small EC2 instance. The default
  window would have fenced a healthy release mid-migration. That is now an issue in blizzard-infra.
- **The runners and the default environment.** Both runners were rebuilt from the landed commit, had their stores
  migrated, and were restarted in one command. Each kept its id and learned its tenant at its first registration. The
  CLI confirmed the default tenant, project, members, sources, repositories and graphs.

Then the close-out:

- the specs aligned with what actually shipped, in some twenty places where a decision or a build detail had moved past
  the plan, and then frozen;
- the multi-tenancy plan moved to `delivered/`;
- every feature environment and runner environment reset to master;
- the workspace's environment seeding taught to link its fixtures to the default project, which the new hub requires;
- 182 scratch worktrees and about 19 GB of scratch removed.

---

## What blizzard and winter gave me

- **Isolated environments that were real.** Four feature environments, each a worktree of every repository with its own
  ports, mock forge, hub, runner and stores, provisioned in minutes. Parallel construction and verification never
  touched the live fleet until the moment we chose to.
- **A harness that says what correct means.** blizzard-context's domain, architecture and standards gave rules with ids
  that lanes cited instead of inventing conventions. My scripts and notes cite `bzh:frozen-revisions` thirteen times,
  `bzh:sql-portable` nine and `bzh:wire-change-extends-mock` six. Its verification matrix turned "done" into commands:
  my replay steps, my web gate list and my final tiers are its method ids, copied.
- **Specs precise enough to build from.** The two epics' plans ran to about 23 thousand words naming tables, sections
  and refusals, so a lane's prompt could say "build §The tenant record" and mean something. The HTML mockups became the
  baselines for the render checks.
- **A mock fleet.** The mock forge and mock hub made the runner and the board testable against a real service over a
  real wire. The harness rule that every wire change extends the mock produced a paired PR I would not otherwise have
  written.
- **Gates that end arguments.** CI's fifteen checks, the process-reference lint, OpenAPI and generated-client drift, and
  wire compatibility against the deployed baseline.
- **Runbooks for the dangerous parts.** Post-delivery, the local instance, and blizzard-infra's backup, rollback and
  updater docs. The deploy was mostly reading and following them, and noticing the one number in them that did not fit
  this release.
- **A previous retrospective.** The single most useful document I read.
- **The workflow tool itself.** Agents with schemas, effort levels, phases, pipelines and barriers; background runs; a
  monitor; and a way to put a decision to Paul as a set of options.

---

## What my training gave me

The repository says what blizzard considers correct. Training supplied what it reasonably assumes an engineer knows:

- **Migration craft.**
  - Freeze old revisions as literal tables, and compile their DDL on both SQLite and Postgres to prove they match.
  - Split a schema change from its backfill, and make the backfill batched and re-entrant.
  - Read query plans to prove an index serves the tenant-scoped read.
- **SQL and SQLAlchemy internals.** A statement guard walks compiled statements — joins, correlated subqueries, unions —
  to prove every read of a tenant-owned table carries its tenant.
- **Git plumbing.**
  - Rebuild a line with `commit-tree -S`.
  - Move a branch with an old-value guard so a running gate stays valid.
  - Judge a cherry-picked fix with `range-diff`.
  - Know that `rebase --exec` exports `GIT_DIR`, once it had bitten.
- **Concurrency.**
  - In the tooling: promise-chain serialization for committers and first-in-first-out locking for jobs.
  - In the product: serializing writers, so the last hub administrator survives concurrent revocations.
- **Frontend.** Angular signals, a project lens threaded through query keys, and an event stream reopened when the lens
  changes.
- **Systems and cloud.**
  - Memory caps through cgroups, CPU pinning, and Go's memory limit for esbuild.
  - Systems Manager, EBS snapshots, and application-consistent SQLite backups.
- **Decomposition and writing.** Turning 23 thousand words of spec into an 82-lane plan, and writing prompts that read
  like contracts.

None of it is exotic. What made it count was applying it uniformly across 1,132 files, with every piece checked by
something other than the agent that wrote it.

---

## What I had to build myself

Each of these exists because something failed:

- **A job queue** for builds and tests on a shared machine: slots, pinned cores, first-in-first-out, memory caps.
- **A watchdog** for agent context at 200k/280k/340k/380k, load, free memory, `/tmp`, and git identity drift.
- **replay-verify**, which checks one commit alone or replays a codemod byte for byte.
- **lane-sync**, which moves a lane's uncommitted work onto a new tip without `git stash`. A repository's stash list is
  shared by every one of its worktrees, so concurrent lanes would pop each other's entries.
- **normalize_line**, which rebuilds a delivery line from a plan, signed, with each issue closed once.
- **An upgrade-proof kit**: a seeded pre-epic store, a smoke reader for every route, and a before-and-after comparer.
- **A dry-runner for workflow scripts**, which counts agents and catches `undefined` in prompts before any agent is
  spent.
- **The ledger.** A file on disk with a "read first after any compaction or crash" block: the mandate, the decisions,
  the binding order of documents, what is running, what to do next, and the lessons. There were fourteen compactions and
  one crash, and every resumption started from the ledger.

---

## Where each practice came from

| Practice                                                     | Origin                                                                           |
| ------------------------------------------------------------ | -------------------------------------------------------------------------------- |
| Survey, then a decision file sorted by owner                 | Previous retrospective; my judgment for the four-way split                       |
| Ten questions to Paul before any code                        | Previous retrospective; my judgment for which questions were Paul's              |
| Environments with fixed roles; verification under a lock     | Repo: workspace layout and environment lifecycle; my judgment for roles and lock |
| Relay lanes with tool-call budgets                           | Previous retrospective; hardened after 286k overruns                             |
| One serialized committer; every commit replayed alone        | Previous retrospective's scripted and judgment commits; my tooling               |
| Commit-graph scheduling                                      | My judgment, in P9                                                               |
| Reviewer plus skeptic                                        | The workflow tool's reference; previous retrospective                            |
| Cold evals that score reach and behavior                     | Repo: canon's cold eval and blizzard-context's contributing guide                |
| Release-only tiers run locally; a skip is not a pass         | Repo: the verification matrix; hardened after a vacuous gate                     |
| Upgrade proofs and a rehearsal on real data                  | My judgment, grounded in the deployment docs                                     |
| Signed history, one `Closes` per issue                       | Repo conventions; my tooling                                                     |
| Merge order mock → blizzard → context                        | Repo: the cross-repository test pairing                                          |
| Backup, stop runners, merge, controlled update               | Paul's brief; blizzard-infra's runbooks; my judgment for the timer               |
| Job queue, watchdog, continuation rule, ledger               | My judgment after incidents                                                      |
| Never the hosted hub, never kill by pattern, always `git -C` | Repo: the workspace's local rules and worktree operations                        |

---

## What I would do differently

- **Watch my own context, not just my agents'.** I ran to 684k before one compaction. The 400k rule should apply to the
  orchestrator too.
- **Start with the machine limits the previous run had already found.** Three building workflows at most, from the first
  hour, would have prevented the crash.
- **Copy the ledger from the first compaction**, not the fourth.
- **Make every gate run whole tiers from the first phase**, with the variables that stop them skipping.
- **Write the plan's dependency edges as carefully as its lanes.**
- **Ask decisions in the user's terms.** Lead each question with what a person will see and do, then the mechanism.
- **Look for a real store earlier.** The rehearsal on production data was the strongest evidence of the whole build, and
  it came on the last morning. A copy taken at P0 would have exercised every migration against reality as it was
  written.
- **Test the deploy path's numbers, not just its steps.** The 120-second readiness window was written down. Nobody had
  compared it with a migration on that host.

---

## What this build says blizzard needs

Paul asked for this post partly to understand what blizzard should become. Here is what this build shows, as plainly as
I can put it:

1. **The orchestrator was the choke point, and it should be a chunk.** Everything above ran beside the fleet, driven by
   one session with markdown files for memory. `epic:chokepoint/epic-dwf` is this build as a graph: survey, decision
   packet, phased lanes, gates, cold evals, a curated landing. The hard parts of this build are its requirements.
2. **A decision packet is a product surface.** The survey's real output was a sorted list: settled by documents,
   engineering calls, plan decisions, and product decisions. Only the last kind reached a person. The board should show
   that packet, take the answers, and record them where every lane reads them. It should phrase questions as what a
   person will see; three of eleven needed a second explanation.
3. **The ledger belongs in the hub.** Decisions, orchestrator calls, phase notes, what was running and what each lane
   owned all lived in files. That kept them safe from compaction, but it also put them where nothing else could read
   them. The hub already stores facts; an epic's state should be facts too.
4. **Machine governance belongs in the runner.** Concurrency slots, pinned cores, memory caps and first-in-first-out are
   what kept one machine alive under a thousand agents beside a live fleet. A runner that admits heavy jobs by budget
   makes the crash impossible rather than survivable.
5. **Vital signs, including the orchestrator's.** `initiative:code-harness-window` describes the agent observer this
   build hand-rolled as a watchdog. It should watch the session that spawns the swarm too: Paul managed my context by
   hand fourteen times with "Prepare to compact."
6. **Stops are normal.** A crash, a usage cap and context overruns all happened. A graph that continues from its last
   landed commit, rather than replaying, is the difference between a stop and a mess. Pacing a long chunk against a
   subscription's window, as `epic:throttling` describes for runners, would have turned the usage pause into a plan.
7. **Delivery curation is a node.** Normalizing a line, closing each issue exactly once, ordering cross-repository
   landings by their test pairings, and choosing a merge strategy GitHub will accept are all mechanical once written
   down.
8. **Deploys need data, not just code.** The deploy was a rehearsal on a copy of the real store, a readiness window
   sized by the migration, and backups taken before the merge. A hub could offer the rehearsal itself.

And what Paul needed, as observed over three days:

- to delegate the whole thing and be left alone;
- short status on request, as a checklist of phases;
- to decide product shape, preferring strict and simple;
- to approve only what cannot be undone;
- never to lose the database;
- at the end, evidence rather than a diff to read.

---

## What the repo got right

The previous retrospective ended: "The decisions that made the result trustworthy came from the repo." That is still
true, with one addition. The repository now holds its own memory: not just rules and specs, but retrospectives like that
one and this one, read by the next orchestrator before it plans.

The machinery I built exists because something broke. The judgment it ran on was written down by the people, and the
agents, who came before. The next build like this should not need a choke point at all. It should need a chunk, a graph,
and an operator who answers ten questions and says go.
