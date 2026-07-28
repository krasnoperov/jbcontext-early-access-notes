# JetBrains Context CLI: early-access field notes

Tested `jbcontext 0.9.5.361` with OpenAI Codex on July 28, 2026.

This is an independent report based on three unrelated private Git
repositories. It contains no repository names, URLs, source code, snippets, or
project-specific architecture.

## Short version

JetBrains Context was easy to install and useful for exploring unfamiliar
code. A behavioural description could lead Codex to the right area without a
filename or symbol.

It did not replace `rg`. The productive workflow was one semantic query,
followed by local source inspection and exact search.

The main concern is the remotely stored vector index and limited visibility
into telemetry.

## Installation and sign-in

The technical installation was simple. JetBrains distributes a native binary,
so no separate runtime, container, or database was required. The official
installer worked on an Apple Silicon Mac.

There was one confusing detail: during a successful installation, the script
printed `Binary: 0B`. It looked like a failed download even though the installed
CLI worked normally.

Authentication used a standard browser flow with license selection. Its final
page was the clearest part of onboarding: it confirmed the selected license,
said the tab could be closed, showed `jbcontext setup-agent`, and explained
what to do from a fresh terminal.

The required EAP agreement created a very different impression. Accepting it is
not merely acknowledging that an early-access binary may be unstable. It also
covers cloud processing, persistent storage of the derived vector index, and
enhanced diagnostic and usage telemetry. That is a significant decision when
the intended input is private source code.

## Configuring Codex

One command registered an MCP server, installed a semantic-search skill, added
managed instructions, and configured hooks and execution rules.

This was technically effective. `jbcontext doctor` passed 9 checks in each
repository; 3 checks for agents not installed were skipped.

The trade-off is visibility. `setup-agent --auto` modifies persistent
user-level configuration and enables background indexing. The CLI can print a
setup plan through additional flags, but this preview would be more useful by
default.

The installed instructions sensibly recommended semantic search only while the
subsystem was unknown, then switching to local reads and exact search.

## Test repositories

The three repositories were indexed independently.

| Repository | Tracked files | Approx. lines | Tracked data |
|---|---:|---:|---:|
| A | 1,535 | 49,895 | 1.9 MB |
| B | 7,228 | 566,291 | 324 MB |
| C | 881 | 56,247 | 2.1 MB |

These measurements exclude dependencies, generated outputs, and caches.

The index is associated with a Git remote and revision, not the complete live
working tree. After Context identifies a file, Codex still needs to read its
current local version.

## First search tasks

The first useful surprise was that behavioural questions could return a
relevant implementation area without a filename, class, or function name.

A focused query worked better than a general architectural topic. For example:

```text
where protected API requests are authenticated before business logic
```

was more useful than:

```text
where is authentication implemented
```

Results included paths, line numbers, snippets, similarity scores, and JSON.
The first few were often enough to identify the subsystem, but still required
verification in the source.

## Context versus `rg`

For a known symbol, local exact search was clearly the right tool:

- `rg`: approximately `0.02 s`;
- JetBrains Context: approximately `1.6 s`.

For one behavioural question, however, a broad regex produced 122 mostly noisy
matches. The first five Context results pointed much more directly to relevant
implementation areas.

The comparison led to a simple working model:

```text
Unknown implementation location
→ one focused Context query
→ inspect the best local result
→ continue with rg, sg, and direct file reads
```

Context helps determine what to search for. Once identifiers and paths are
known, `rg` is faster, exact, local, and better suited to finding every usage.

## Evidence from Codex history

A sanitized review of 35 recent coding sessions found:

- 34 used `rg`;
- 26 contained at least five `rg` operations;
- median pre-edit exploration was 7 search or inspection operations, rising to
  17 at the 75th percentile and 30 at the 90th;
- only 7 initial prompts already named a file or path.

This suggests an opportunity to reduce the first several blind searches. It
does not imply that Context can replace tests, exact usage checks, local
verification, or reading the implementation.

## The misleading part of `jbcontext analyze`

I did not understand the scope of `jbcontext analyze` immediately. Running it
inside a repository made the report look repository-specific, and the projected
reductions initially looked like measured local savings.

Repeating the command from three repositories on the primary development host
returned the same result. The current working directory does not limit the
dataset, and the tested CLI has no repository filter for Codex history.

The output reported `wiredIn: false`, described its framing as `projected`, and
referenced `ultimate-swebench`. Context was not present in the analyzed
sessions. The reported reductions were therefore modelled estimates, not a
local before-and-after measurement.

The methodology was not documented in enough detail to validate how local
history became projected savings. A clean A/B test with the same tasks would be
more convincing.

The JSON can expose project paths, sizes, session counts, token and cost
estimates, and agent activity. It requires sanitization before publication.

## Cloud index and telemetry

According to the
[JetBrains Context CLI EAP Agreement](https://www.jetbrains.com/legal/docs/terms/jetbrains-context-cli-eap/),
repository data is parsed locally and transmitted to JetBrains. The resulting
vector index is persistently hosted in JetBrains' cloud database on Google
Cloud Platform.

The index may include embeddings, code-structure representations, chunk
summaries, extracted comments, and metadata. JetBrains states that raw
plain-text source is excluded from persistent cloud storage after indexing and
that this data is not used to train generative models.

This differs materially from a local-first tool that keeps embeddings and the
vector database on-device.

Local files recorded search/indexing events and analytics-upload activity. We
could not inspect the uploaded payload, and found no CLI command to inspect or
disable telemetry.

A local-index mode and transparent telemetry controls would materially change
the decision for private repositories.

## Final impression

The search was more useful than expected. It solved a narrow but common
problem: reaching the right part of an unfamiliar codebase before the agent
knows the project's vocabulary.

Installation was easy, browser onboarding was excellent, and the Codex
integration encoded a sensible hybrid workflow. The rough edges were mostly
about transparency: automatic configuration, `analyze`, and the boundary
between local and cloud data.

The most accurate summary is:

> Semantic discovery first, exact local verification afterwards.

The technical value is already visible. Trust and observability need to catch
up with it.
