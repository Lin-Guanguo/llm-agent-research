---
description: Case note on a Dayfold structured-output index drift incident and why business validators plus deterministic repair are necessary.
last_updated: 2026-05-21
---

# Dayfold LLM Index Repair Case

## Source

Conversation/session reference: `codex://threads/019e49dd-f33f-7950-8c6c-16b1473507b3`

This note records the architecture lesson, not the full debugging transcript.

## Case Summary

The observed symptom was that two adjacent downstream pages became misaligned:

- The upstream editor output still contained two different page contents.
- The downstream page enhancer produced output that made those pages appear duplicated or shifted.

The root cause was not the upstream editor. It was a structured-output coordinate-system error in the downstream enhancer.

The runtime requested enhancement for absolute page indices in the current snapshot:

```text
requested = [1, 2, 3, 4, 5, 6]
```

The model returned page indices as if the batch were locally numbered:

```text
returned = [0, 1, 2, 3, 4, 5]
```

The old defensive logic only realigned when the returned set had no overlap with the requested set. In this case there was partial overlap (`1..5`), so the runtime accepted the wrong indices. The result was an off-by-one shift: one page consumed the next page's enhanced result, and the final page fell back to the original content.

## Lesson

Schema validation only proves output shape. It does not prove business semantics.

A Pydantic model can guarantee that `page_index` is an integer and that a list of enhanced pages exists. It cannot guarantee that the model used the correct business coordinate system unless the runtime validates that explicitly.

This case supports a three-layer output boundary:

| Layer | Responsibility |
|---|---|
| Schema validation | Check field presence, types, and structural constraints. |
| Business validator | Check semantic invariants such as requested index set, duplicates, omissions, and bounds. |
| Deterministic repair policy | Repair only errors that are provable from the local contract. |

The important distinction is that repair must be narrower than validation. Validation can detect many bad outputs; repair should only modify outputs when the runtime can prove the intended mapping.

## Correct Repair Shape

The stable policy is:

```text
if returned_index_set == requested_index_set:
    accept as absolute indices
elif returned == [requested_i - 1 for requested_i in requested]:
    repair as deterministic off-by-one relative indices
else:
    reject / drop / fallback / retry
```

This treats the specific `12345 -> 01234` pattern as a deterministic repair case. Other failures should not be guessed:

- missing pages
- duplicate pages
- mixed absolute and relative indices
- unrelated out-of-range indices
- reordered results with ambiguous intent

Those should go through retry, fallback, drop, or explicit failure depending on the node's safety policy.

## Implementation Detail

Repair order matters.

The runtime must normalize the model-returned index into the target business index before constructing the final business object. Otherwise the visible text may be remapped correctly while inherited metadata still comes from the wrong original page.

The risky order is:

```text
wrong index -> construct EnhancedPage -> remap key
```

The safer order is:

```text
model index -> normalize target index -> construct EnhancedPage from target page
```

This matters for hidden fields such as character references, inspiration ids, provenance, or other fields inherited from the source page.

## Architecture Implication

This is a concrete example of why a production structured-output agent needs runtime-side guardrails.

For Dayfold-style agents, the LLM should be allowed to produce candidate structure, but the runtime must own:

- business validation
- deterministic repair
- fallback / retry policy
- telemetry for repaired and rejected outputs

This fits the larger Dayfold architecture thesis: low-token, high-stability business agents cannot rely only on the model obeying prompts. They need programmatic contracts around every model-produced artifact.

