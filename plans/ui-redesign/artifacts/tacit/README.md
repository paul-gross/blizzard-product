# Tacit: the fleet proceeds on tacit consent

## Thesis

Every fleet UI so far, the baseline included, asks the operator to read state and then decide whether to act. With 10 to
80 agents running overnight, that has it backwards. Nearly everything the fleet does is fine, and the operator's real
job is to say *no*, rarely and precisely. Tacit inverts the arrangement. The screen never shows state as state. It shows
**motions**: each is something the fleet *will* do, on a visible **fuse**, unless the operator objects. When the fleet
cannot pick a default, the motion has a **blank** in it that only the operator can fill. When the operator objects to
the same kind of motion twice, the objection becomes a **standing order**, and the fleet carries it out without asking
again. Silence means consent. The operator's skill shows up as a screen that grows *emptier*: the better your standing
orders, the fewer motions reach you. The question the screen answers is not "what is the fleet doing?" but "what will
happen if I do nothing, and when do I next need to look?"

## Principles

1. **Future tense only.** The fleet writes every line as an intention: "C-PTP2 *will* land PR #1492". Past events go to
   a record. Present state appears only as evidence, unfolded under a motion.
2. **Silence is consent.** A motion fires when its fuse burns out. The operator does not have to approve anything for
   the fleet to keep moving.
3. **Blanks are the only things that wait.** A needs-human chunk is a sentence with a hole in it ("Shall I ____?"),
   offering 1–3 named fills. Blanks have no fuse and sit above everything else.
4. **Weight by what is held, not by status.** Blanks are ordered by what they block downstream. C-6J9H holds 23 chunks;
   C-QWYS holds none and says so. The baseline gave both the same red card. Tacit tells you that one can wait until
   morning.
5. **Two inks, and colour means authorship.** Graphite is the fleet's hand. Vermilion is the operator's: strikes, holds,
   signatures, blanks, and standing orders. No colour is used for status. Anything red on the screen is either your will
   or a place that needs it.
6. **The operator's own acts are motions too.** Retire a host, revoke a token, or enrol one, and the act is tabled with
   a 30-second fuse in your ink. You can strike it before it fires. Undo is part of the grammar, not a confirmation
   dialog.
7. **Always forecast the walk-away.** A standing panel (desktop) or headline (mobile) states what happens by morning if
   you leave now, and what would change that.

## The grammar of operator acts

There are seven verbs. Every one works on a motion or a blank, and every one is undoable with `u`.

| Act            | Key         | Pointer             | Touch                      | Effect                                                                                                                            |
| -------------- | ----------- | ------------------- | -------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| consent now    | `space`     | click, then `space` | n/a (silence does this)    | Fires the motion early. On an order-struck motion, overrules the order once.                                                      |
| object         | `x`         | margin              | swipe left                 | Strikes the motion in vermilion. The chunk returns to waiting.                                                                    |
| object always  | `X`         | margin              | desk only                  | Opens a composer pre-filled with the fleet's generalisation of the motion. `enter` makes it a standing order, `esc` objects once. |
| hold           | `h`         | margin              | desk only                  | Freezes the fuse in your ink until you release it.                                                                                |
| fill a blank   | `1` `2` `3` | click the option    | tap the option             | Writes your answer into the hole.                                                                                                 |
| take over      | `t`         | margin              | "send takeover to my desk" | Copies the resume command. On a phone, it sends the command to the desk instead.                                                  |
| leave for desk | n/a         | n/a                 | "leave for the desk"       | Mobile only: defers a blank to the desk docket, held.                                                                             |

Capacity verbs on the purse and hands view: `p` stays a runner (it takes nothing new and finishes what it holds),
`-`/`+` set fewer or more hands, `$` raises the allowance, and `R`/`T`/`E` table retire, revoke or enrol with a fuse.
Navigation: `j`/`k` move, `enter` unfolds the evidence (node trail with failures marked, runner, PR, graph, cost), and
`u` undoes. Clicking a standing order lifts it, and clicking again restores it.

**Standing orders** are written in the operator's own sentence form, for example "Object to a third landing after two
identical failures". Each one shows how many times it fired today. Motions an order struck still appear on the docket,
already struck, labelled with the order that struck them. You see your past will at work, and `space` overrules it once.

## The four surfaces

- **Docket (desk board).** The masthead is a posture sentence, not a KPI strip. Below it come the blanks, then the next
  hour of motions sorted by fuse, then a dashed horizon line summarising everything further out in one paragraph. The
  right column holds the standing orders, the walk-away forecast, your hand (a log of every mark you made), and
  everything that lapsed while you were away.
- **Purse and hands (desk fleet).** The purse is the daily allowance plus each subscription window, drawn as 60 ticks:
  filled for what is drawn, a caret for time elapsed, and a vermilion stroke at your ceiling. The hands are one cell per
  environment slot under each runner, with a vermilion edge where a hand waits on you. The capacity motions follow
  ("r-claude will put its next free hand on C-1DTX"). Host facts are evidence in the aside, not a call to act.
- **"Do you need me?" (mobile board).** This is a different contract. The headline answers the question in one line
  ("Yes, twice." or "No. Put it away."), followed by the time until you are next needed. The heaviest blank fills the
  screen, and each option shows its consequence ("frees 23 at once", "23 stay idle"). Below that sits a short list of
  what is lapsing, where a left swipe objects. There is no tab bar.
- **Purse and brakes (mobile fleet).** The phone gets only the coarse levers: raise or lower the allowance, stay a
  runner, and lift a standing order. Its headline is the one capacity fact that matters tonight ("The purse runs dry at
  17:40").

## Scaling from 8 running to 80

The docket's length is bounded by **time**, not by fleet size. It shows the blanks plus the motions due within the
horizon (an hour), and everything beyond is folded into one paragraph. At 80 agents, three mechanisms keep it short:

- **Like motions merge.** "r-claude will claim 9 survey runs as slots free" is one motion, and one objection covers all
  nine.
- **Standing orders absorb repetition.** A pattern the operator has judged once never surfaces again. It appears only as
  a fire count.
- **Blanks stay few.** Even at 80 agents, an operator faces a handful of blanks, and stake ordering puts the one that
  matters first.

The hands view grows in rows of slot cells. Each runner gets one line per workspace, so 80 hands across six runners fit
in about 14 rows.

## What it deliberately drops

- **The status columns.** Backlog, ready, waiting, running and done are the fleet's bookkeeping, not the operator's
  decisions. Status shows up in a motion's verb and in evidence.
- **The 53 ready cards.** Ready chunks appear only when they are about to be claimed. Chains appear as stakes ("holds
  23").
- **The KPI strip.** It is replaced by a posture sentence that ends in a time.
- **The activity feed.** It is replaced by the record of what lapsed and by your hand.
- **Status colour.** There is none. Colour means authorship.
- **Tabs** for graphs, events, gardening and admin. They stay reachable through evidence and other routes, but they are
  out of the operator's loop.

## Risks

- **Forecasts must be honest.** Every fuse and the walk-away paragraph are predictions. If "on pass, at about 13:22" is
  routinely wrong, the operator stops trusting the fuse, and the paradigm collapses back into reading state. Event-
  driven motions are labelled with their trigger, not a fake clock, for this reason.
- **Consent by silence can launder mistakes.** A motion the operator never saw still fires. The mitigation is that the
  lapsed record is first-class and stake-ordered, and that standing orders let the operator push risk categories behind
  a signature (§1: "Hold every delivery into winter for my signature").
- **Standing orders become a policy language.** They can drift, conflict or go stale. Fire counts help, but a real build
  needs order conflict detection and expiry.
- **Prose density.** Sentences cost more width than tokens. The mock compensates with two-line motions and a
  semi-condensed face, but at 1280px or less the margin would have to fold into a hover.
- **Debugging needs state.** Tacit hides state behind evidence, so a harness engineer chasing a wedged runner needs the
  evidence unfold to be deep and fast, or a plain state route beside it.

## Files

- `board-desktop.html` / `.png`: the docket. Fuses burn live, and C-PTP2 lapses into the record at about 1:40. Try
  `j`/`k`, `space`, `x`, `X` then `enter`, `h`, `1`–`3`, `t`, `enter` and `u`. Append `#keys=j,j,j,j,X,Enter` to replay
  keys.
- `fleet-desktop.html` / `.png`: purse and hands. Try `$`, `p` on a runner, `-`/`+`, `R` then `x` to strike your own
  motion, and `enter`.
- `board-mobile.html` / `.png`: "Do you need me?". Tap an option, swipe a lapsing motion left, or use "leave for the
  desk".
- `fleet-mobile.html` / `.png`: purse and brakes. Use the allowance steps, "stay it", and the order switches.

The type is Noto Sans (semi-condensed) for the fleet's voice, Noto Serif italic for the operator's hand, and Noto Sans
Mono for ids. All three are named system fonts, and the mocks make no network requests.
