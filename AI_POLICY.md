# Generative AI policy

**This repository contains software used for in vitro diagnostic testing of human clinical
specimens. Generative AI must not be used to write code for it, and specimen-derived data must not
be given to a generative AI system.**

This applies to everyone and to every tool: coding agents, IDE assistants and inline completion,
commercial chat interfaces, API calls, and locally hosted models. It applies whether the output is
committed directly or pasted, adapted, or retyped by a person first.

**Not permitted**

- Generating or modifying any code, configuration, pipeline definition, schema, or report template
  in this repository with a generative AI system.
- Giving specimen-derived data to a generative AI system — reads, classifier output, taxon tables,
  QC reports, identifiers, run metadata, logs containing them, or draft clinical reports.
- Asking a general-purpose AI system to interpret, triage, or sanity-check a run. Cerebro's own
  `cerebro-ciqa` module is a documented exception with its own conditions of use — see the README
  disclosure and `AGENTS.md` §4.4. The distinction is between a versioned, configuration-recorded
  component of the system and an unrecorded chat session.

**Permitted, and encouraged**

- Using AI to read, explain, review, and criticise this codebase.
- Using AI to draft documentation, SOPs, validation and verification plans, risk analyses, issue and
  PR text, and other prose.
- Using AI to plan work and to learn the surrounding technologies in general terms.

**Why:** provenance and traceability for a regulated device; the risk of silent semantic drift in
diagnostic logic that current test coverage would not catch; the confidentiality of patient data;
and the requirement that clinical interpretation be attributable to a named competent person.

**Full policy, rationale, regulatory context, and the process for lifting the restriction:**
[`AGENTS.md`](AGENTS.md). Where this summary and `AGENTS.md` differ, `AGENTS.md` governs.

Questions: see §12 of `AGENTS.md`.