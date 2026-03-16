# agent-jumpstart

[OpenAI Tokenizer](https://platform.openai.com/tokenizer)

## Readable (Canonical) Versions

GPT-4o Estimate:
- Tokens: 1,732
- Characters: 8,687

| File | Tokens | Characters |
|---|---|---|
| `project-agnostic-AGENTS-readable.md` | ~1,737 | 8,710 |
| `agnostic-agent-service-AGENTS-readable.md` | ~3,454 | 17,322 |
| **Combined** | **~5,191** | **26,032** |

## RDF Triple Versions (Compact)

Rules are expressed as RDF-style subject–predicate–object triples to minimize token consumption.
See `project-agnostic-AGENTS-rdf.md` and `agnostic-agent-service-AGENTS-rdf.md`.

### Triple notation

```
<subject> <PREDICATE> <object> [; qualifier]

generateText MUST_SET maxSteps explicitly ; NEVER omit|Infinity
transient CLASSIFY_AS networkTimeout|502|503|rateLimit → retry
tool.execute MUST_NOT fillMissingFields|mergeCalls|formatForModel
```

`|` = OR, `,` = AND/list, `;` = qualifier, `→` = maps to, `+` = compound value.

GPT-4o Estimate (RDF versions):

| File | Tokens | Characters |
|---|---|---|
| `project-agnostic-AGENTS-rdf.md` | ~800 | 4,014 |
| `agnostic-agent-service-AGENTS-rdf.md` | ~1,264 | 6,341 |
| **Combined** | **~2,064** | **10,355** |

~60% token reduction vs. the original prose versions (measured 2026-03-15 against GPT-4o tokenizer).
