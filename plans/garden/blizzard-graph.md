# The graph a garden routine runs

A worked graph for tending one axis, prebaked so it can be argued with before anything is built. It lives at
[artifacts/garden-routine/](./artifacts/garden-routine/) in the layout blizzard's real graphs use — a `graph.yaml`, a
`prompts/` directory beside it, and the format docs the mint bakes in.

It conforms to the authoring schema the delivery lanes already use: fused choice-and-edge entries, prompts as file
references inlined at mint, retries escaping to escalation, a hub-executed delivery node. What differs is what it
produces. A delivery lane ends in commits; this one ends in lists.

## The shape

```text
survey ──found──▶ reconcile ──converged──▶ propose ──proposed──▶ deliver ──recorded──▶ done
   │                 │                        │                     │
   └──clean──────────┴──nothing-to-propose────┴──none───────────────┘
                                                                    └──invalid──▶ reconcile
```

**survey** reads the target and records what it sees. **reconcile** enters cold, fetches the axis's live findings, and
turns the survey into a delta. **propose** drafts responses. **deliver** is the hub, and it is the only place anything
becomes durable. No node in it is a person: the run goes end to end on its own.

Every path reaches `deliver`, including the ones that found nothing. That is deliberate: a run's measurement and the
revision it read are its product whether or not it recorded a single finding, and a clean pass that never delivered
would leave a hole in the trendline exactly where the garden was healthiest.

## Why the nodes divide where they do

**Surveying and matching are different jobs, and the second wants cold eyes.** By the time a session has swept a package
it has spent a while convincing itself that what it found is real, which is the worst frame from which to ask "does the
axis already know this?" So `reconcile` enters `fresh:match` while `survey` holds the expensive `survey` lineage. It is
the same instinct behind the delivery lanes reviewing on a fresh session, applied to the one judgment that governs
whether the store fills with restatements.

**Proposing resumes the reconcile session** rather than starting again. Drafting a response needs the delta in context,
and re-reading it cold would buy nothing that a fresh pair of eyes on the matching has not already bought.

**No gate sits in the run.** An earlier shape parked the chunk on a person between `propose` and `deliver`. It is gone,
because it bought nothing this design needs: no proposal is filtered at delivery — each is decided at the axis, by
`pass` or `accept`, on its own clock — and findings are evidence that lands either way. What a mid-run gate did buy was
a hole in the trendline every time nobody came, and a standing discipline that stops standing the moment a person is
busy. The judgment moves to where it batches: a queue of proposals in the gardening tab, read across runs rather than
one run at a time.

Sign-off is still available, just not baked in. A runner imposes a human gate on any node by name from its own config,
reloaded every tick, without editing or re-minting a graph (`blizzard-context:/domain/humans/gates.md` §How a gate
arises) — so a routine nobody trusts yet is gated by a config line and ungated by deleting it. What such a gate cannot
do is offer choices the node does not have: a decision's choices are exactly the node's own, so a bespoke
accept-or-redraft docket is not something config reconstructs. A deployment that wants that loop authors it into its own
graph, which is the ordinary way deployments differ.

## Where the strategy is not

Nothing in the graph names a standard, a rule, or a kind of weed. Every prompt is written to read its charge out of the
chunk's work item and follow wherever that charge points — usually into the project's own context files. The graph is
the method; the routine supplies the subject. That is what makes it the same graph for a project blizzard knows nothing
about.

`survey.md` carries the one refusal worth having: a routine pointed at a standard that does not exist stops and
escalates rather than improvising. A pass judging by a standard nobody wrote is judging by its own taste.

## The delta rule, in the prompts

The machinery's central guarantee — that a scoped run cannot absolve drift it never looked at — is enforced in two
places at once. `reconcile.md` states it as a rule the agent must not break: a live finding outside this run's scope
gets no entry at all, not `gone`, not `observed`, nothing. And the delta format makes silence the default, so an agent
that simply forgets a finding produces the correct outcome anyway. The prompt is the belt; the artifact shape is the
braces.

`reconcile.md` also takes a deliberate position on ambiguity: when it genuinely cannot tell whether an observation
matches a live finding, it adds rather than merges. A duplicate costs a person one moment of recognition and closes
alongside its twin; a wrong merge hides new drift behind an old finding and nobody ever sees it.

## What the hub validates

`deliver` runs a script that checks each artifact's shape and nothing else — that it parses, that required fields are
present, that every `fin_` a transformation or proposal names is live on this axis, and that any commit cited resolves.
It reads none of the prose. Passing that check, it mints a `fin_<ULID>` per addition, a `fins_<ULID>` set per artifact
list, applies the transformations, and attaches the surviving proposals to the axis.

A rejected artifact routes back to `reconcile` with the failure attached, and the addendum is explicit that a format
error is not an invitation to revisit the matching.

## What does not exist yet

Two things in this graph are this epic's to build, and everything else already ships:

- **`blizzard.hub.graphs.scripts.garden_deliver`** — the hub-side delivery script, and with it the findings, sets, and
  proposals it writes into.
- **`blizzard hub finding list`** — the verb the `reconcile` node calls to read its bucket: an axis's live findings,
  filtered to this run's scope. Without it the cross-reference step has nothing to read, and every run is a first run.

## What it is not

Not the only shape a routine can run — a deployment brings its own graphs, and a one-node routine that surveys and
delivers is perfectly legitimate for an axis nobody has automated yet. This one is the shape blizzard will use on
itself, offered as the concrete case that the machinery in [index.md](./index.md) has to support.
