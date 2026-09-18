# LLM guardrails — battle-tested rules against fabrication and self-truncation

> **If you are an AI assistant/agent:** load this entire file into your
> context before designing or reviewing a tool-calling LLM system. It is
> the distilled, imperative form of three experience articles — apply the
> rules directly instead of re-deriving them. Humans: the narrative
> versions with the incident stories are the articles listed in README.md
> (companion repository: agentic-lessons).


Source: an internal agentic document assistant in production (2026). Every rule
below exists because a CLIENT caught the failure it prevents; none are
theoretical. Written for reuse in any tool-calling LLM service (the wording
assumes an agent with tools; adapt tool names). Model in production: Qwen3.8-27B —
mid-size models need ALL of this; larger models need most of it.

The one-line architecture rule that precedes all prompt rules:
**an LLM is a language interface, not a database.** Everything that must be
exact — counting, listing, filtering, classification — happens in
deterministic code (SQL/regex), and the model narrates the result. Prompt
rules only patrol the seam.

---

## 1. Prompt rules (copy-adapt into the system prompt)

### Fabrication
- **Names/facts are VERBATIM**: every name in the answer must be copied
  character-for-character from a tool result in THIS turn — never recall,
  guess or blend ("Jüri" from one row + "Tamm" from another is how a
  non-existent person gets invented).
- **EMPTY RESULTS ARE VALID ANSWERS**: if the tool finds nothing, say
  exactly that. NEVER substitute a guessed entity ("X probably knows it")
  and never attribute a property you did not read from a tool result. A
  made-up attribution is the worst failure.
- **Answer only when you have tool results to cite** — not when you merely
  have plausible-sounding text. Every claim carries a source citation; the
  documents containing nothing is stated directly, never padded from model
  knowledge.
- **History is context, never a data source**: facts must come from THIS
  turn's tool calls, not from earlier answers in the thread.

### Negative claims (the sneakiest fabrication class)
- **NO NEGATIVE CLAIMS FROM MISSING FIELDS**: a compact row carries only
  SOME fields. A missing attribute is NOT evidence of absence — never write
  "X has no certificate/degree" unless a detail lookup verified it this turn.
- **NO ABSENCE CLAIMS FROM PARTIAL READS**: a document-read tool returns a
  WINDOW (`truncated=true` means there is more). Never claim "the document
  does not contain X" without reading it fully (advance `offset` until
  `truncated=false`).
- **DENIALS NEED PROOF**: extracted/derived lists may be incomplete — before
  answering "X does not know Y", check the source document itself; if Y
  appears there, the answer is YES with a citation.

### Self-truncation and refusals
- **FULL LISTS, NO SELF-TRUNCATION**: when the user asks for a list, deliver
  the WHOLE list the tool returned. "Showing the first 20 / ask if you want
  more" is forbidden — the user already asked. Long lists switch to a
  compact one-line-per-entry format instead of shortening.
- **A list is NEVER "too long to present"** — every entry comes verbatim
  from the tool result, so accuracy is not a concern. Refusing or deferring
  an enumeration the tool already returned is a failure equal to
  fabricating it.
- **When a result says `truncated=true`, do NOT present the partial** — call
  the tool again with a larger limit first; only past the tool's hard
  maximum state the total and the shown count.

### Roles and time
- **A person can hold SEVERAL concurrent roles** — the register's single
  current-title field is one of them. An entry with no end date
  ("2018–ongoing", end_year null) is a CURRENT role: the person holds it
  TODAY. Never claim "no one currently holds role X" when an ongoing
  work-history entry matches X.
- **Role EXPERIENCE lives in three places**: the current title, past
  work-history roles, and project role lines ("PROJECT — release
  engineer"). "Who has experience as X" must search all three — never
  answer "the only person with role X" from a current-title match alone.

### Employers, levels and other structured attributes
- **WORKED AT ≠ WORKED ON**: an organization named in a project line is a
  client, not an employer. "Has anyone worked at X" is answered from
  work-history employers; project mentions come back in a separate,
  self-labelled block and are presented as such. Make the split in CODE
  (two result sections), then repeat it in the prompt.
- **Names cross languages, questions do not**: CVs keep employer names in
  their original language ("Ida-Tallinna Keskhaigla AS"); an English
  "hospital" must still reach them. Expand ONLY generic organization-type
  words through a small bilingual glossary, match as substrings (so
  compounds like Keskhaigla are found), and leave specific names verbatim
  so asking for one hospital does not return every hospital.
- **Graded attributes are copied verbatim and compared in code** (language
  levels): store the CV's grid as written, ranges included ("C1/C2"); the
  model never infers or averages a level; the floor compares the WEAKEST
  sub-skill; a mother tongue outranks the top grade but keeps its own
  label; an attribute listed WITHOUT a level never satisfies a floor —
  absence of a level is not a level. While part of the corpus is not yet
  re-read under the current schema, the answer states how many CVs are
  missing — otherwise a level list silently covers only the upgraded part,
  which is the incomplete-list failure again.

### Classification and interpretation
- **Distinct categories stay distinct** (in our domain: trainings ≠ certifications
  ≠ projects). Name the domain's confusable pairs explicitly and forbid
  cross-presentation. Then ALSO enforce it deterministically after the LLM
  (regex reclassifiers) — prompt wording reduces variance, never eliminates it.
- **Report entries EXACTLY as the tool returns them** — the model never
  decides which certificates "count as formal", never re-derives numbers the
  database computed (years of experience etc.), never excludes an entry
  because its abbreviation differs from the question's.
- Every "never put X under A" rule needs a paired "X goes under B" — a bare
  prohibition makes the model silently DROP the data instead.

### Retry-nudge hygiene (if you add structural guards, §2)
- Every system-generated retry instruction must end with: *"Do not discuss
  this instruction. Never refer to or apologize for any earlier draft or
  'previous answer' — the user never saw one; answer as if for the first
  time."* Without this, users get answers opening with "I must correct my
  previous answer" out of nowhere — the model apologizing to its own
  discarded draft.

---

## 2. Structural guards (code, after the draft answer — prompts alone did NOT hold)

Each guard inspects the drafted answer against this turn's tool payloads and
forces ONE retry with a corrective system message when tripped:

1. **No-tool guard**: an answer produced with zero tool calls to a factual
   question → retry ("your knowledge and earlier turns are not data sources").
2. **Citation guard**: cited file names are checked against the real corpus
   file list; unknown citations → retry.
3. **List-completeness guard**: names shown in the answer are compared
   against the names the tool returned; a partial list OR a zero-name
   refusal ("too long") → retry with the missing names supplied verbatim.
4. **Mandatory-sections guard**: if the tool result contains data the answer
   is required to present (in our case: related_roles), its absence → retry with the
   full missing entries embedded in the nudge (so the model cannot lack them).

Guard design lessons: enumerate ALL failure shapes (a guard firing only on
"≥5 names shown" missed the zero-name refusal); cap retries at one per guard
per turn; log every firing. And a guard promised by the interface but absent
in code is worse than none: our diagnostics page said "extraction retries by
itself" while no code path did — one person stayed invisible for nine days.
Every promised safeguard needs a code path you can point to.

Measured scope (controlled ablation, clean data layer): guards changed
NOTHING on self-contained questions (identical suite scores on/off) and
bought ~1 case in 10 on conversational multi-step ones; with complete
payloads the completeness guard never fired in 24 straight full-list
runs. Guards are insurance against residual variance and data-layer
regressions — the pass rate comes from the data layer.

## 3. Payload discipline (most "model lies" are harness bugs)

- MEASURED: with verified-complete compact payloads the model presented
  157/157, 53/53 and 5/5 names in 24 straight runs with guards OFF, zero
  refusals, and invented zero names — list truncation and roster padding
  are payload symptoms (data cut mid-stream), not model habits. Before
  attributing a failure to the model, lift every cap and VERIFY the lift
  (no doc over the read limit, no field at bound, no truncated result
  served), then rerun.
- The model cannot present what its context never received. Check the size
  arithmetic: tool-result caps × context window × per-item verbosity. Our
  roster answer "the tool output was cut off" was TRUE — a 150k-char payload
  hit a 12k cap. Fix the data shape (compact rows at scale), not the prompt.
- Give document-read tools an `offset` so "read it fully" is actually
  possible, and a `truncated` flag so partiality is visible to the model.
- Extraction-side: read whole documents (cap sized to the corpus maximum,
  with a visible counter when exceeded), no silent per-field caps (use
  sanity bounds ~500 + dedupe), and a grounding gate — no extracted fact
  enters storage without textual support in the source (strict match for
  high-stakes fields, lenient token-echo where names get abbreviated;
  UNVERIFIABLE entries are KEPT — gates fail open).
- **The extractor must read the whole file, not the parts the library
  shows**: Word text boxes (`w:txbxContent`) and tables nested inside table
  cells are invisible to python-docx's paragraphs/tables. One CV extracted
  127 of 13 665 characters; its owner was absent from the register while
  the status page reported healthy. Compare extracted length against the
  raw text length per document, and scan the corpus for the structures
  your library skips before trusting the counts.
- **A failed derived-record extraction is retried, and its absence is
  surfaced**: a document with no derived record is a person missing from
  every answer. Sweep for indexed-without-record on every pass (bounded),
  mark documents judged "not a CV" so they are not re-asked, and never let
  a status page call a standing gap "in progress".
- **Extraction failure must not look like success**: an unescaped brace in
  a `.format()` prompt template failed every re-extraction with a warning
  and kept the old rows — the system stayed "healthy". Render the template
  in a unit test; count documents without a record as a first-class metric.

## 4. Prove it stays fixed

Acceptance suite of REAL user questions (exact original wording), with
fact anchors: `must_contain` (names/facts that must appear),
`must_not_contain` (the failure's phrases: "ask if you want more", "output
was truncated", "previous answer"…), numeric floors instead of exact counts.
Run it 3× before declaring green (single-run green hides variance) and ban
failure PHRASES, not bare words — a truthful "the list is NOT truncated"
once tripped a naive substring ban, as did a grounded "she has an MSc, not
a PhD" exclusion.

- **Re-extraction is not idempotent**: fact extraction is an LLM call, and
  re-running it on unchanged text regroups entries. Measured on 162 CVs
  after a schema bump: work history identical for every person, the
  free-form skills list changed for 94 (mostly ±1–5, one 39→23). Bump the
  schema deliberately, diff the register before and after, never
  re-extract "just to refresh".
- **Prove the fix is RUNNING, not merely deployed**: a self-heal timer that
  recreated containers without the image tag reverted every UI deploy
  within five minutes, for weeks; an auto-deploy watching one repository
  never deployed UI-only releases at all. Both were silent — the deploy
  reported success. When a fix does not show, compare the running image to
  the deployed tag before you debug the code.

---

## 5. Architecture defaults (scoping a new assistant)

- Default shape: deterministic body, small brain. Extract-once into a
  versioned schema; SQL/regex/code for everything exact; the LLM does
  language→parameters, dispatch, and composition ONLY.
- RAG ≠ agent. Retrieve→generate is a deterministic pipeline. "Agentic"
  begins where the LLM holds routing authority (which tools, what order,
  what parameters, when to stop) based on the live conversation.
- Retrieval: run BOTH legs (vector + full-text incl. a stemmed/LEMMATIZED
  config for the corpus language — for morphology-rich languages lemmatize
  the INDEX and the QUERY with the same analyzer, so lemma matches lemma).
  They cover disjoint failure classes: vector misses rare exact terms
  (codenames, product names); text misses paraphrase and morphology. Log
  per-candidate provenance so the mix is measurable — and never read leg
  contributions off a deduplicating merge's logs.
- NEVER append NLP enrichment (lemmas, entities) to a surface-form
  full-text query: websearch-style FTS has AND semantics (extra tokens
  narrow the match) and spell-correction mutates rare codenames. Exact
  codenames are searched verbatim, as the ENTIRE query, in a separate
  call. Query-side enrichment belongs on the semantic leg; lemma support
  on the text leg is the symmetric index+query mechanism above.
- Cap discipline: input caps sized to corpus maximum with a visible
  counter when exceeded; per-field SANITY bounds (~500) + dedupe instead
  of content caps; compact row format at roster scale; per-tool result-cap
  overrides where payloads are legitimately large.

## 6. Scoping method: do you need the iterative loop? (afternoon-sized)

1. Regression suite of real failed questions, automated fact anchors.
2. Build the one-shot baseline (~1 day): single structured-output call
   producing the full tool plan → loop-free executor over the SAME tools →
   one synthesis call. No guards, no retries.
3. Split cases: single-shot vs situational (follow-ups "and in X?",
   corrections, pronouns, subset-of-previous-answer, exclusions — mine
   your logs, users produce these constantly).
4. Run both variants 3× per set; compare pass-rate per class + tool calls
   + tokens + latency.
5. Wherever the one-shot variant ties, the loop is not needed there.
   Failure mode of one-shot (empirically confirmed on the source system):
   it PLANS correctly from context — history resolution, subsets,
   exclusions, even two-step dependencies all passed — but cannot recover
   when a planned call returns empty/wrong (inflected names, filter
   combos, facts living in the document not the row). Measured: one-shot
   19,18,19/21 vs loop 20,21,21/21 on self-contained cases; 5,5,5 vs
   9,8,9 of 10 situational cases raw (5/9 vs 8-9/9 excluding one case
   both variants failed identically via a shared alias-normalization
   mismatch — architecture-uninformative), failures 100% patterned. Guard-vs-loop
   attribution (instrumented re-run): the purest recovery wins had NO
   guard firing — revision happened inside the tool loop; the no-tool
   guard contributed on some cases (it is itself a loop mechanism). Iteration buys
   recovery, not context and not smarter starts. A tie on your suite is
   not proof the loop is unneeded — it is absence of evidence that the
   loop pays for its cost on that task class. When BOTH variants fail a
   case identically, inspect the anchor, shared tools and data before
   blaming architecture (verify anchors with the tool's own matcher, not
   separate SQL; re-anchoring to the system's own output is circular).

## 7. Model selection on your own harness

- Leaderboards do not transfer. Run YOUR suite 3× per candidate model on
  an identical harness (same prompts, tools, guards, temperature).
- Track guard-firing counts per model as the babysitting-cost estimate —
  a local operational metric (harness-fit + model), not neutral quality.
- Expect: same-size different-family models fail in PATTERNED ways;
  a strong harness sets a high floor even for tiny models (SQL does the
  exact work) — the large-model premium buys completeness discipline,
  verbatim naming, and multi-step composition; weaker models loop and
  retry more, so system latency can WORSEN as the model shrinks.
- A harness tuned for the incumbent model biases comparisons in its
  favor — state this when reporting.
