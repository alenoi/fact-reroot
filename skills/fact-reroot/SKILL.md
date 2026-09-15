---
name: fact-reroot
description: Re-verifies a specific stored claim from a clean, uncontaminated context instead of arguing with the context that produced it — the fix for context poisoning, memory poisoning and hallucination snowballing, where a wrong conclusion entered memory once and now every cross-examination routes back to it as though it were gospel. Use whenever a memory file, digest, glossary entry or CLAUDE.md assertion is suspected wrong, or the assistant keeps defending a prior answer instead of re-deriving it. Trigger on "biztos ez így van?", "honnan tudod ezt?", "ellenőrizd a forrását", "nézd meg a gyökerét", "kutasd újra", "ez tényleg így van?", "szerintem ez tévedés", "hol ered ez az info", "fact-reroot", "re-verify this", "trace this claim to its source", "where did this come from", "recheck from scratch", "de-poison", "verify from the root". Not for a plain lookup where nothing is actually in doubt, general web research with no stored claim behind it, or ordinary code debugging.
---

# fact-reroot

A wrong conclusion that enters memory once gets cited at every step after, and arguing
with it inside the same context does not work: Zhang et al. (2023) found GPT-4 identifies
87% of its own incorrect claims when each one is put to it alone in a fresh session, yet
keeps elaborating on them once it is committed to an earlier answer — the model
prioritises consistency with what it already said over truth ("How Language Model
Hallucinations Can Snowball", arXiv:2305.13534). That is context poisoning / memory
poisoning — ASI06 in OWASP's Top 10 for Agentic Applications — doing its work.
Cross-examination inside the contaminated context cannot fix it, because the poisoned
premise is itself an input to every subsequent answer. The fix is not a better argument;
it is re-deriving the fact somewhere that has never seen the stored conclusion, then
comparing.

Run these seven steps in order. Steps 2 and 4 are a read-only fan-out and a blind
execution, and both belong in sub-agents rather than in your own context: step 2 is a
wide sweep whose transcript you do not want, and step 4 only means anything in a context
that has never seen the stored conclusion. Step 5 is verification, and stays with you.

## Optional configuration

The skill runs with no configuration at all. With none, step 2 discovers where records
live and step 3 discovers where evidence lives, using the kinds listed under each step. A
config file does not add a capability; it replaces guessing with knowing, in an
environment where you already know.

Look for `fact-reroot.config.json` in this order:

1. `./.claude/fact-reroot.config.json` — project-local
2. `~/.claude/fact-reroot.config.json` — user-level

If both exist, union the array entries with the project-local ones listed first; for the
scalar keys the project-local file wins. If neither exists, fall back to discovery.

```json
{
  "recordStores": [
    { "path": "<absolute path to a dir or file where stored claims live>",
      "kind": "agent-memory | instructions | digest | notes | docs | tickets",
      "autoLoads": true,
      "note": "optional free text" }
  ],
  "primarySources": [
    { "path": "<absolute path>",
      "kind": "repos | notes | docs | data",
      "git": true,
      "note": "optional free text" }
  ],
  "quarantineDir": "<absolute path step 4 moves an auto-loading record aside to>",
  "blindCheck": { "agents": 2, "checkExternalLiterature": true }
}
```

Those four keys are the whole schema; nothing else is read. The shape is the method's own
spine written down. `recordStores` are the **witnesses** — the records that assert the
claim, which step 2 traces. `primarySources` are where the **evidence** lives — the
artefacts step 3 reads, and step 5 reads again itself. `autoLoads` is the flag that
decides whether step 4's blindness comes free or has to be bought: a record that loads
itself into every session, a sub-agent's included, cannot be hidden from a fresh agent by
wording the brief carefully.

## 1. Scope one claim

Turn what's being questioned into a single falsifiable statement — a sentence that is
either true or false, not a topic. If the user handed you a broad area ("check what we
know about X"), enumerate the candidate claims living under it and either pick the one
that's actually in doubt or ask which one, when it's genuinely ambiguous which claim
is meant.

Run one claim per fact-reroot. A run that tries to settle three claims at once returns
mush you can't adjudicate — you won't be able to tell which claim the blind agent's
answer actually confirms or refutes.

## 2. Trace the record (read-only fan-out)

Find every place the claim appears, and build the provenance chain — which record cites
which, in what order.

If a config is present, `recordStores` names the places to search: search those first and
search them exhaustively. Otherwise discover them. A stored claim lives in one of these
kinds of place, and you look in all of them this environment actually has:

- **Persistent agent memory** — the directory this assistant writes memory into, plus its
  index file (`MEMORY.md`, or whatever the harness calls it). Locate it rather than assume
  it: check the harness settings for a memory-directory key, then the conventional
  locations under `~/.claude/`.
- **Auto-loading instruction files** — `CLAUDE.md` at every level that applies (user,
  project, subdirectory), and its siblings from other tools: `AGENTS.md`, `GEMINI.md`,
  `.cursorrules`, `.github/copilot-instructions.md`. Search these before the rest. A claim
  living here is re-asserted at the top of every session, which is both why it spreads and
  why step 4 will have trouble with it.
- **Cached digests and summaries** — session summaries, handover notes, daily or weekly
  digests, anything that compressed earlier work into standalone assertions.
- **Personal note vaults** — Obsidian, Logseq, a wiki, a folder of markdown. Grep the
  whole vault, not the notes you expect; a claim's second home is usually a note nobody
  remembers writing.
- **Project documentation** — `README`, `docs/`, architecture decision records, a
  Confluence or Notion export, the repo wiki. ADRs weigh more than the rest here: they are
  written to be cited later, so a wrong one propagates well.
- **Source repos and their git history** — `git log -S` on a distinctive phrase to find
  the commit that introduced the claim's wording, `git log --diff-filter=D -- <path>` when
  the claim points at a file that no longer exists, `git blame` on the line itself, and the
  commit messages, which often carry the reasoning the code never did.
- **Issue trackers** — issues, pull request descriptions, review comments. A claim
  frequently originates in a comment someone wrote once and everything since has cited.

This is a sweep across unknown locations — a Grep/Glob fan-out. Delegate it to read-only
sub-agents (one per store, in parallel, where the stores are independent) rather than
reading everything yourself.

Follow the chain back to its root — the earliest record, the one nothing else cites.
Then classify that root as one of two kinds:

- **Observation** — a file and line, a commit hash, a verbatim quote, a measured
  figure. Something that points at a primary artefact.
- **Inference** — derived, reasoned, summarised, with no primary source underneath it.

Name explicitly what you're hunting for: an inference that has been repeated so many
times, in so many downstream records, that it now reads like a fact. That's the
poisoning mechanism, and step 2 is where you catch it in the act.

Then check one further property of every record you found: **does it auto-load?**
Instruction files, memory index files and the agent-memory directory are pulled into every
session, a sub-agent's included, and recalled memories arrive as system-reminders. A store
marked `autoLoads: true` in the config counts as one without further checking; for a
discovered store, decide it by asking whether a brand-new session in this environment
would see that file without being pointed at it. If the claim lives in one of those, a
fresh sub-agent cannot be blind to it however you word the brief, and step 4 will look
like it succeeded while proving nothing. Mark those records — step 4 has to treat them
differently.

## 3. Identify the primary evidence

The primary evidence is the artefact the claim is *about* — the actual file, line,
commit, document or URL — not the memory entries and digests that assert it. Those are
witnesses, not the event.

If a config is present, `primarySources` names where evidence lives, and an entry with
`git: true` is one whose history counts as evidence in its own right, not just its working
tree. With no config, the primary evidence is wherever the claim's subject actually is:
the repo for a claim about code, the vendor's own documentation for a claim about an API,
the invoice or the pricing page for a claim about a price, the dataset for a claim about a
number.

If step 2's root was classified as an observation, this is usually the same artefact —
confirm it still says what the root claims. If the root was an inference, look for
primary evidence independently of the inference; don't let the inference tell you
where to look.

If no primary evidence exists anywhere — nothing to point at, only records asserting
records — that absence is itself the finding. Carry it into step 5 as UNSUPPORTED; do
not manufacture a source to fill the gap.

## 4. Blind re-derivation — the load-bearing step

Spawn a fresh sub-agent — new context, nothing carried over from this session — and
give it only:

- the question, phrased as if no one has ever answered it before
- pointers to the primary evidence from step 3 (paths, commit hashes, URLs)

**Hard prohibition:** it must not be told the stored conclusion, must not be told that
a stored conclusion exists at all, and must not be pointed at the memory files,
digests or index entries that carry it. Not "and by the way we think it's X" —
nothing. An agent that "helpfully" folds the prior conclusion into its brief has
silently destroyed the method, and the run will still look like it succeeded, because
of the same 87%-vs-committed effect this skill exists to route around — the moment the
agent sees the prior answer, it starts defending it instead of deriving it.

The principle is not this skill's own: Chain-of-Verification withholds the original
response from its verification prompts for exactly this reason (Dhuliawala et al.,
arXiv:2309.11495). What is specific here is applying it to a claim stored in a previous
session rather than to an answer being drafted in the current one.

**If step 2 flagged the claim as living in an auto-loading file**, blindness is not
available by default and you have to buy it. In order of preference: quarantine the
entry first — move it aside into the config's `quarantineDir`, or any directory nothing
auto-loads from, and do not delete it, you still need it to compare against — then run the
blind check against an environment that no longer asserts it; or run the check somewhere
those files do not load at all, a scratch directory outside every project whose
instruction files carry the claim; or, failing both, run it anyway and record in the
step 7 report that the re-derivation was not blind. A non-blind re-derivation
that agrees with the stored claim is worth close to nothing, because agreement is what
a poisoned context produces anyway. Say so rather than banking it.

For claims about something public or external (a library's behaviour, a vendor's
pricing, a documented API), add a second blind check against current primary
literature or documentation, not against what your own records already say — that is what
`blindCheck.checkExternalLiterature` asks for. Where two independent blind agents are
cheap to run, run both, on the same brief, and compare what they return before you move to
step 5; `blindCheck.agents` says how many this environment wants.

## 5. Adjudicate — yours, never delegated

Compare the blind result(s) against the stored claim, having read the primary evidence
yourself — not just the sub-agent's summary of it — for anything that will feed a
decision or ship to the user. Issue exactly one verdict:

- **CONFIRMED** — independent re-derivation matches, and you've read the primary
  evidence yourself. If the re-derivation was not blind, it cannot carry this verdict
  on its own; the primary evidence has to be enough by itself.
- **REFUTED** — re-derivation contradicts the stored claim.
- **UNSUPPORTED** — no primary evidence either way; the claim was always an inference.
  Do not report this as REFUTED — an unsupported claim might still be true, it's just
  never been checked against anything.
- **STALE** — it was true when written and no longer is: the file moved, the flag
  changed, the price changed, the repo got restructured. This is the common case for
  memories about paths, flags and prices — check it before reaching for REFUTED.

The first three follow the standard three-way vocabulary of fact verification —
SUPPORTED / REFUTED / NOTENOUGHINFO, after FEVER — and STALE is the addition.

If the two blind agents disagree, that's not settled by picking the more confident
answer or averaging — read the primary evidence yourself and decide.

## 6. Repair and propagate — propose first

Once you have a verdict, walk the provenance chain from step 2 forward, not just fix
the root. A corrected root with three uncorrected downstream copies still re-poisons
the next session that reads one of the copies.

For every record you touch, attach provenance: what the source was, when it was
verified, and whether it's an observation or an inference. That's what stops the next
fact-reroot from having to redo this work.

Propose the diff and get the user's go-ahead before writing to any record store — never a
silent edit to a memory file, an instruction file or a note vault, all three of which are
read by future sessions that will not know you changed them. Producing the diff is a
sub-agent's job — whatever document-editing or general-purpose agent this setup has, on a
cheap tier; deciding to apply it is not.

## 7. Report short

State, in this order: the claim as tested; the verdict; the root and how it was
classified; what primary evidence was actually read (paths, lines, commits); what was
changed (or proposed); what still needs a human. Skip everything else — the transcript
of how you searched belongs in the sub-agent's return, not in this report.

## Guards

- Never re-derive inside the context that carried the claim. That's the one thing this
  skill exists to prevent — a fresh context is not optional.
- A fresh sub-agent is not automatically a clean context. Auto-loading files reach it
  too, so establish in step 2 whether the claim is in one before trusting step 4.
- "Found nothing" in step 2 is a result to interrogate (wrong root? wrong search terms? a
  store this environment has that the config never listed?), not confirmation that the
  claim is clean.
- The config is a shortcut, not an authority. A path in it that no longer exists is a
  finding about this environment, and a store it fails to list is still in scope —
  discovery is the floor, the config only raises it.
- If blind agents disagree, the parent reads the evidence — never averages, never
  defers to the more confident-sounding one.
- Never invent a source to close a gap in step 3. No primary evidence found means
  UNSUPPORTED, and nothing else.
- Tiering: step 2's fan-out and step 4's blind re-derivation belong on a cheaper, faster
  model than the one you are orchestrating with — a pure grep-and-list trace needs the
  cheapest tier available. Step 5's adjudication runs on your own model and is the one
  piece of this that stays with you. The rule is the split, not any particular model
  name: send the work as low as you can still check the output of.
