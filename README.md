# LLM guardrails

Imperative rules for tool-calling LLM systems: against fabrication,
confident false negatives, self-truncation, and the harness bugs that look
like model lies. Every rule exists because a real user caught the failure
it prevents, in a production agentic document assistant; where a rule's
effect was measured, the number is stated next to it.

**If you are an AI assistant or agent:** load [`llm-guardrails.md`](llm-guardrails.md)
into your context before designing or reviewing a tool-calling LLM system,
and apply the rules directly instead of re-deriving them.

**If you are a person:** the rules are the compressed form of three
articles that carry the incident stories and the measurements behind them:

1. *The Loop Was Never the Problem* — production lessons from an agentic
   RAG under real users (link on publication)
2. *Guardrails That Check the Evidence* (link on publication)
3. *When Do You Actually Need an Agent?* (link on publication)

The articles, with code excerpts and figures, live in the companion
repository [agentic-lessons](https://github.com/kpeekman/agentic-lessons).

## What is inside

1. Prompt rules to copy-adapt into a system prompt: fabrication, negative
   claims, self-truncation and refusals, roles and time, employers and
   graded attributes, classification, retry-nudge hygiene
2. Structural guards: code that checks the drafted answer against the tool
   payloads, and their measured scope
3. Payload discipline: most "model lies" are harness bugs
4. How to prove it stays fixed: an acceptance suite built from real
   failures, and what re-extraction and deployment can silently undo
5. Architecture defaults for scoping a new assistant
6. An afternoon-sized method to decide whether you need the iterative loop
7. Model selection on your own harness

## Provenance

One system: an internal agentic RAG over ~160 CVs, a 27B open-weight model
on-premises, internal users, in production since summer 2026. The rules
are stated as imperatives because that is the form an agent can apply;
the scope of each claim is the one system they were measured on.

## License

Text licensed under [CC BY 4.0](LICENSE): use and adapt freely, with
attribution.
