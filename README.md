# fact-reroot

A Claude Code skill that re-verifies one specific stored claim from a clean,
uncontaminated context, instead of arguing about it inside the context that
produced it.

## The problem

An agent writes a conclusion into some form of memory — a memory file, a
digest, a glossary entry, a `CLAUDE.md` assertion, a note in a ticket. The
conclusion is wrong, or it was right once and the world underneath it moved.
From that point on, every later question that touches the same topic gets
answered by an agent whose context already contains the wrong conclusion as
an apparent fact. Asking the agent to double-check does not help: the
poisoned premise is an input to the very reasoning that is supposed to catch
it, so the agent re-derives the same wrong answer, now with more apparent
confidence than before. This is Memory & Context Poisoning — ASI06 in the
OWASP Top 10 for Agentic Applications, published by the OWASP GenAI Security
Project (2026 list) — and repeated over several turns it produces
hallucination snowballing: each pass treats the last pass's error as settled
ground.

Arguing inside the contaminated context does not fix this, because the
contaminated context is what is doing the arguing. The fix is to re-derive
the fact somewhere that has never seen the stored conclusion, and compare
the two independently reached answers.

## What it does

`fact-reroot` takes one doubted claim, finds every place it is recorded,
identifies the primary evidence the claim is actually about, hands a fresh
sub-agent the question and pointers to that evidence without telling it what
the stored answer is, and adjudicates the result. If the claim turns out to
be wrong or outdated, it proposes the fix across every place the claim was
recorded — it never edits silently.

### The seven steps

1. **Scope one claim** — turn the doubt into a single falsifiable statement.
   One claim per run; a vague worry does not get re-derived, a precise one
   does.
2. **Trace the record** — a read-only fan-out that finds every place the
   claim appears and builds the provenance chain back to its root. That root
   is classified as an observation (it points at a primary artefact) or an
   inference (derived, with no primary source underneath), and each record
   found along the way is flagged for whether it auto-loads into a session.
3. **Identify the primary evidence** — the artefact the claim is *about*,
   not the memory entries that assert it. A note saying "the config lives at
   X" is a witness; the actual file system state at X is the event.
4. **Blind re-derivation** — the load-bearing step. A fresh sub-agent gets
   the question and pointers to the primary evidence, and is never told the
   stored conclusion or that one exists.
5. **Adjudicate** — never delegated. The orchestrating session compares the
   stored claim against the blind re-derivation and reaches exactly one
   verdict.
6. **Repair and propagate** — walk the whole provenance chain found in step
   2, not just the root, and propose a diff for every stale record. Nothing
   is edited without that proposal being shown first.
7. **Report short** — the verdict, the evidence, and the proposed diff. Not
   a transcript of how the sub-agent got there.

### The four verdicts

| Verdict | Meaning |
|---|---|
| `CONFIRMED` | The blind re-derivation matches the stored claim; the primary evidence still supports it. |
| `REFUTED` | The blind re-derivation contradicts the stored claim; the primary evidence says something else now. |
| `UNSUPPORTED` | The claim traces back to an inference with no primary evidence underneath it. Not `REFUTED` — it might still be true, it has just never been checked. |
| `STALE` | The claim was true when recorded but the primary evidence has since moved or changed; the record is outdated rather than wrong on arrival. |

## Why a fresh sub-agent is not automatically a clean context

The obvious objection is "just ask a new sub-agent." That is necessary but
not sufficient. A newly spawned agent still auto-loads whatever the
environment loads for every session — a global instructions file, an
auto-memory index, a project `CLAUDE.md` or `AGENTS.md`. If the doubted claim
lives in one of those auto-loading files, a sub-agent spawned in the same
environment inherits it before the first prompt is even written, and step 4
looks like it succeeded — a plausible, independently worded answer comes
back — while actually proving nothing, because it was never blind. Step 2's
`autoLoads` flag exists precisely to catch this. Where it is true, blindness
is not free: the entry has to be quarantined (moved aside for the duration
of the check) or the check has to run somewhere that file is not loaded.

## Prior art, and what this actually adds

Almost none of the parts here are new. Blind verification, provenance chains,
propagating retraction, a three-way verdict on a claim against evidence — each
has a literature behind it, parts of it decades old.

The blindness in step 4 is Chain-of-Verification's. In the factored variant,
Dhuliawala et al. answer each verification question in a prompt that withholds
the original draft: "Those prompts do not contain the original baseline
response and are hence not prone to simply copying or repeating it" (§3.3).
Step 4's hard prohibition was published in 2023.

The provenance machinery is older. Doyle's truth maintenance system (1979) and
de Kleer's assumption-based variant (1986) separated premise nodes from derived
ones, recorded the justification behind every belief, and propagated a
retraction through everything derived from it. Step 2's
observation-versus-inference split and step 6's walk down the chain are that
structure in markdown. AGM belief revision gave retraction its formal
semantics; its belief-base versus belief-set distinction cuts in the same place.

The verdicts are borrowed too. FEVER (Thorne et al., 2018) fixed the field's
vocabulary for adjudicating a claim against evidence as SUPPORTED / REFUTED /
NOTENOUGHINFO; three of the four verdicts here rebadge it, and only STALE sits
outside the scheme.

Shipped agent-memory frameworks do detect contradictions between stored records
and resolve them — by recency, by weighting sources for reliability, by
cryptographic ancestry, or by giving an LLM judge both versions. None goes back
to the artefact the claim is about. Eywa, a provenance-grounded memory
architecture, says as much about itself: "Provenance establishes source support,
not external truth... world-level truth verification remains outside the memory
layer."

What is left unoccupied is small. Blind re-derivation applied to cross-session
persistent memory rather than to an answer generated in flight, since CoVe has
no store and its problem ends when the run does. The auto-load consequence:
Anthropic documents that a non-fork sub-agent inherits the whole `CLAUDE.md`
hierarchy at startup, but no one appears to have drawn the verification
consequence from it — blindness is unavailable by default and has to be bought.
And STALE as a fourth verdict folded into a support/refute scheme; temporal
validity is studied, but as its own task.

Whether that gap is worth filling is not this repository's call. A survey of
memory for autonomous LLM agents lists among its open challenges "External
validation (check reflections against ground truth when available), uncertainty
quantification (decay confidence over time without confirming evidence),
adversarial probing (periodically challenge stored beliefs with
counterexamples)" (§9.3), and names no reviewed system that checks stored
memories against external primary evidence.

One caveat, on step 6. Attaching provenance to a corrected record assumes a
later reader uses it, and the evidence for that assumption is not encouraging.
In a study of 26 researchers given a claim-evidence interface over LLM-generated
scholarly text, granular provenance "significantly lowered participants' trust
compared to the baseline", yet "this increased caution did not translate to
behavioral changes" — they kept relying on the output regardless. Step 6 makes
a record honest. It does not make the next reader careful.

References: Dhuliawala et al., "Chain-of-Verification Reduces Hallucination in
Large Language Models", arXiv:2309.11495 (2023), ACL Findings 2024 · Doyle, "A
Truth Maintenance System" (1979) · de Kleer, assumption-based TMS (1986) ·
Thorne et al., FEVER (2018) · "Memory for Autonomous LLM Agents",
arXiv:2603.07670 · Eywa, arXiv:2605.30771 · Martin-Boyle et al., "PaperTrail: A
Claim-Evidence Interface for Grounding Provenance in LLM-based Scholarly Q&A",
CHI 2026, arXiv:2602.21045.

## Worked example

A memory file says a project's log parser lives at `src/parsers/log.py`.
Asked to check a fix there, the assistant hesitates: the file layout sounds
off. `fact-reroot` scopes the claim, traces it to that one memory entry
(an observation, not auto-loading), and sends a blind sub-agent to find the
parser with only the repository as evidence, no path given. The sub-agent
reports `src/parsing/log_parser.py`; the file was moved in a refactor eight
months ago. Verdict: `STALE`. Proposed repair: update the memory entry to
the new path.

## Installation

As a marketplace plugin:

```
/plugin marketplace add alenoi/fact-reroot
```

then install the `fact-reroot` plugin from that marketplace.

Manually, without the plugin system: copy `skills/fact-reroot/` into
`~/.claude/skills/fact-reroot/` (or a project's `.claude/skills/`) so the
directory contains `SKILL.md` directly.

## Configuration

The skill works with no configuration at all — by default it discovers
record stores and primary sources by searching the working environment. An
optional config file lets it skip that discovery and go straight to known,
concrete paths.

File name: `fact-reroot.config.json`. Lookup order: project-local
`./.claude/fact-reroot.config.json`, then user-level
`~/.claude/fact-reroot.config.json`. If both exist, the `recordStores` and
`primarySources` arrays are unioned (project-local entries first) and the
project-local file wins for scalar keys (`quarantineDir`, `blindCheck`).

| Key | Type | Meaning |
|---|---|---|
| `recordStores` | array | Places a claim might be recorded — the witnesses. Each entry: `path` (absolute), `kind` (`agent-memory`, `instructions`, `digest`, `notes`, `docs`, or `tickets`), `autoLoads` (boolean — whether this path is loaded into every session automatically), `note` (optional free text). |
| `primarySources` | array | Places the evidence the claim is *about* actually lives. Each entry: `path` (absolute), `kind` (`repos`, `notes`, `docs`, or `data`), `git` (boolean — whether it is a git repository), `note` (optional free text). |
| `quarantineDir` | string | Absolute path step 4 moves an auto-loading record aside to, so a blind sub-agent spawned afterward does not inherit it. |
| `blindCheck` | object | `agents` (how many independent blind sub-agents to run) and `checkExternalLiterature` (boolean — for a claim about something public or external, whether to also run a second blind check against current literature or documentation, not just the configured sources). |

See `examples/fact-reroot.config.example.json` for a realistic mixed setup —
an agent-memory directory, a global instructions file, a digest cache, a
note vault, and a source repository, spanning both Windows and POSIX paths.

## Trigger phrases

The skill's frontmatter triggers on both English and Hungarian phrases —
"biztos ez így van?", "ellenőrizd a forrását", "re-verify this", "trace this
claim to its source", "de-poison", and similar. To add another language,
edit the `description` field in `skills/fact-reroot/SKILL.md`'s frontmatter
and add the equivalent phrases; no other file needs to change.

## License

MIT. See `LICENSE`.
