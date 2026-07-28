# JetBrains Context CLI: early-access observations

Tested `jbcontext 0.9.5.361` with OpenAI Codex on July 28, 2026.

This independent report contains no repository names, URLs, source code,
snippets, or project-specific details.

## Test data

Three unrelated private Git repositories were indexed separately.

| Repository | Tracked files | Approx. lines | Tracked data |
|---|---:|---:|---:|
| A | 1,535 | 49,895 | 1.9 MB |
| B | 7,228 | 566,291 | 324 MB |
| C | 881 | 56,247 | 2.1 MB |

The measurements exclude dependencies, generated files, caches, screenshots,
and other ignored or untracked data.

`jbcontext doctor` passed 9 checks; 3 integrations not installed on the test
machine were skipped.

## Setup

The browser confirmation page after authentication was excellent: it confirmed
the selected license, said the tab could be closed, and showed the exact next
commands.

The remaining setup was less transparent. `setup-agent --auto` modifies
persistent agent configuration, installs hooks, and enables background
indexing.

## Search observations

For a known symbol:

- `rg`: approximately `0.02 s`;
- JetBrains Context: approximately `1.6 s`.

For one behavioural question, a broad regex returned 122 mostly noisy matches.
The first five Context results pointed to relevant implementation areas.

Context was useful when the implementation location was unknown. Once a
relevant file or symbol was found, direct file reads and `rg` were faster and
more precise.

## Codex history

A sanitized review of 35 recent coding sessions found:

- 34 used `rg`;
- 26 contained at least five `rg` operations;
- median pre-edit exploration: 7 search or inspection operations;
- 75th percentile: 17;
- 90th percentile: 30;
- only 7 prompts already named a file or path.

This suggests an opportunity to reduce initial exploration, but it is not an
A/B measurement of Context.

## `jbcontext analyze`

The analyzer was run against two local Codex histories:

| Dataset | Tasks | Exploration share | Exploration per task | Projected token reduction | Projected tool-call reduction | Projected cost/turn reduction |
|---|---:|---:|---:|---:|---:|---:|
| Primary development host | 1,298 | 5% | ~9 s | ~20% | ~13% | ~13% |
| macOS test host | 134 | 14.09% | 109.9 s | 21.24% | 13.45% | 14.16% |

Running the analyzer from three different repositories on the primary host
returned the same result. The current working directory does not limit the
dataset, and the tested CLI has no repository filter for Codex history.

The output reported `wiredIn: false` and described the reductions as projected.
Context was not present in the analyzed sessions, so these are modelled
estimates rather than measured savings. The CLI referenced
`ultimate-swebench`, but did not provide enough methodology to validate the
projections independently.

## Data handling

According to the
[JetBrains Context CLI EAP Agreement](https://www.jetbrains.com/legal/docs/terms/jetbrains-context-cli-eap/),
repository data is parsed locally and transmitted to JetBrains. The resulting
vector index is persistently stored in JetBrains' cloud database on Google
Cloud Platform.

The hosted index may contain embeddings, code-structure representations, chunk
summaries, extracted comments, and metadata. JetBrains states that raw
plain-text source is excluded from persistent cloud storage after indexing.

Local logs also recorded analytics uploads. We could not inspect the uploaded
payload, and no telemetry inspection or opt-out command was discoverable in
the CLI.

This is materially different from tools that keep embeddings and the vector
database entirely on-device. A local-index mode and transparent telemetry
controls would make Context easier to adopt for private repositories.

## Conclusion

JetBrains Context was useful as a semantic entry point into unfamiliar code:

> Semantic discovery first, exact local verification afterwards.

The main unresolved issues are remotely stored derived code data, limited
telemetry visibility, and `analyze` projections that cannot yet be
independently verified.
