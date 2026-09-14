# The graph blizzard's routines run

A worked graph for tending one axis, prebaked so it can be argued with before anything is built. It lives at
[artifacts/garden-routine/](./artifacts/garden-routine/) in the layout blizzard's real graphs use — a `graph.yaml` with
a `prompts/` directory beside it.

It conforms to the authoring schema the delivery lanes already use: fused choice-and-edge entries, prompts as file
references inlined at mint, retries escaping to escalation, a hub-executed delivery node. What differs is what it
produces. A delivery lane ends in commits; this one ends in lists.

## The shape

```text
survey ──found──▶ reconcile ──converged──▶ propose ──proposed──▶ deliver ──recorded──▶ done
   ├──excessive──▶   │                        │                     │
   └──clean──────────┴──nothing-to-propose────┴──none───────────────┘
                                                                    └──invalid──▶ reconcile
```

**survey** reads the target and records what it sees — or bails out, per §The short circuit below. **reconcile** enters
cold, fetches the routine's live findings for this run's scope, and turns the survey into a delta. **propose** drafts
responses. **deliver** is the hub, and it is the only place anything becomes durable. No node in it is a person: the run
goes end to end on its own.

Every path reaches `deliver`, including the ones that found nothing, for the reason [machinery.md](./machinery.md)
§Delivery gives: a clean pass that never delivered would leave a hole in the trendline exactly where the garden was
healthiest.

## The short circuit

`survey` has a third way out. Before it enumerates anything it takes a gut-check — could this scope be inventoried well
within one context, or would recording what is here exhaust it? — and when the answer is the second it takes the
`excessive` choice: one finding, class `excessive-scope`, and no inventory at all. What a person does with that is
[blizzard-garden.md](./blizzard-garden.md)'s to say; what the graph owes is the path.

That path still runs through `reconcile` and `propose`, deliberately. Routing it straight to `deliver` would save
nothing worth having — the expense being avoided is the enumeration, and a one-entry delta costs almost nothing to match
— while giving up the only thing that makes a bail-out repeatable. Without `reconcile`, a weekly routine pointed at
overgrown ground would mint a fresh `excessive-scope` finding every week instead of observing the one it already
carries, and the bail-out would become its own kind of accretion. The choice is kept separate from `found` so the run's
history records which way it went, and an operator can ask which runs bailed rather than inferring it from a
suspiciously short delta.

## Why the nodes divide where they do

**Surveying and matching are different jobs, and the second wants cold eyes.** By the time a session has swept a package
it has spent a while convincing itself that what it found is real, which is the worst frame from which to ask "does this
routine already know this?" So `reconcile` enters `fresh:match` while `survey` holds the expensive `survey` lineage. It
is the same instinct behind the delivery lanes reviewing on a fresh session, applied to the one judgment that governs
whether the store fills with restatements.

**Proposing resumes the reconcile session** rather than starting again. Drafting a response needs the delta in context,
and re-reading it cold would buy nothing that a fresh pair of eyes on the matching has not already bought.

**No gate sits in the run.** An earlier shape parked the chunk on a person between `propose` and `deliver`. It is gone,
because it bought nothing this design needs: no proposal is filtered at delivery — each is decided at the routine, by
`pass` or `accept`, on its own clock — and findings are evidence that lands either way. What a mid-run gate did buy was
a hole in the trendline every time nobody came, and a standing discipline that stops standing the moment a person is
busy. The judgment moves to where it batches, in the gardening tab.

Sign-off is still available, just not baked in: a runner imposes a human gate on any node by name from its own config,
so a routine nobody trusts yet is gated by a config line and ungated by deleting it. What such a gate cannot do is offer
choices the node does not have — a decision's choices are exactly the node's own, so a bespoke accept-or-redraft docket
is not something config reconstructs. A deployment that wants that loop authors it into its own graph, which is the
ordinary way deployments differ.

## Where the strategy is not

Nothing in the graph names a standard, a rule, or a kind of weed. Every prompt is written to read its charge out of the
chunk's work item and follow wherever that charge points — usually into blizzard's own context files, per
[blizzard-garden.md](./blizzard-garden.md). The graph is the method; the routine supplies the subject. That is what
makes it the same graph for a project blizzard knows nothing about, and why a deployment can adopt it wholesale.

One class is the exception, and it earns the exception: `excessive-scope`, the finding a survey records when the ground
in front of it is past what a pass can hold. That is a fact about the run rather than about the target — the same
judgment on the same evidence, whatever is being tended — so it belongs to the method, and the prompts name it outright.
Naming it is also what lets it converge: `reconcile` can only match this run's bail-out against the last one's if both
spell it the same way. The response class the bail-out earns is left to the strategy, because proposals are never
matched across runs and nothing breaks if a deployment calls it something else.

`survey.md` carries the one refusal worth having: a routine pointed at a standard that does not exist stops and
escalates rather than improvising. A pass judging by a standard nobody wrote is judging by its own taste.

## Where the delta rule is enforced

The machinery's central guarantee — that a scoped run cannot absolve drift it never looked at — is enforced in two
places at once. `reconcile.md` states it as a rule the agent must not break: a live finding outside this run's scope
gets no entry at all, not `gone`, not `observed`, nothing. And the delta format makes silence the default, so an agent
that simply forgets a finding produces the correct outcome anyway. The prompt is the belt; the artifact shape is the
braces.

`reconcile.md` also takes a deliberate position on ambiguity: when it genuinely cannot tell whether an observation
matches a live finding, it adds rather than merges. A duplicate costs a person one moment of recognition and closes
alongside its twin; a wrong merge hides new drift behind an old finding and nobody ever sees it.

The wire formats the prompts write to are the platform's, not this graph's — [machinery.md](./machinery.md) owns them,
and the `artifacts:` map names the shipped format docs rather than restating them. A rejected artifact routes back to
`reconcile` with the failure attached, and the addendum is explicit that a format error is not an invitation to revisit
the matching.

## What does not exist yet

Everything in this graph ships today except the two pieces [machinery.md](./machinery.md) names as this epic's to build
— the hub-side delivery script with the findings, sets, and proposals it writes into, and the CLI verb the `reconcile`
node calls to read its bucket. Without the second, the cross-reference step has nothing to read and every run is a first
run.

## What it is not

Not the only shape a routine can run. A deployment brings its own graphs, and a one-node routine that surveys and
delivers is perfectly legitimate for an axis nobody has automated yet. This one is the shape blizzard will use on
itself, offered as the concrete case the machinery has to support.
