# OpenAI JSON Pricing

[OpenAI JSON Pricing](https://icoretech.github.io/openai-json-pricing/pricing.json)
is a public machine-readable OpenAI pricing catalog offered by iCoreTech, Inc.
It is generated from the official OpenAI pricing page and published as static
JSON for services that need automated pricing refreshes.

Prices are public API USD rates for reporting estimates. They do not measure a
subscription charge, provider invoice, or quota debit. Subscription statements
about long-context fees do not establish an API pricing cutoff or make API rates
zero.

## Canonical URL

[https://icoretech.github.io/openai-json-pricing/pricing.json](https://icoretech.github.io/openai-json-pricing/pricing.json)

## Catalog schema

The catalog uses `schema_version: 2`. Its root contains `generated_at`, `source`,
`source_url`, `models`, `models_count`, `tools`, and `tools_count`. The root
`generated_at` is the only generation timestamp; model records and rule provenance
do not repeat it. Read inventory counts from the catalog itself.

Models with generic token pricing include `context_pricing` with rule `version: 1`
and a `tiers` object. Its keys exactly match the model's importable token-price
tiers. Rules apply separately to each model and tier; there is no global cutoff.
Modality-specific, tool, and per-minute descriptors retain their own pricing
shapes and do not gain a generic token rule.

For example, this synthetic model's `context_pricing` value defines a standard
tier cutoff. The fingerprint illustrates the required format, not real source
evidence:

```json
{
  "version": 1,
  "tiers": {
    "standard": {
      "status": "known",
      "long_context_input_tokens_gt": 500000,
      "provenance": {
        "locator": "gpt-example-a/standard/context-note",
        "evidence_sha256": "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"
      }
    }
  }
}
```

Each tier descriptor has exactly the fields for its status:

| Status | Fields besides `status` and `provenance` | Meaning |
| --- | --- | --- |
| `known` | `long_context_input_tokens_gt` | A positive integer cutoff: input at or below it uses the lower band; input strictly above it uses long-context pricing. |
| `uniform` | None | No numerical cutoff applies. This requires no `long_context` price row and no explicit source condition requiring a cutoff. |
| `unknown` | `reason`, `observed_rules` | Source evidence cannot establish a rule. Consumers must report the estimate as unavailable, not zero. |

The cutoff counts total input, including cached reads and cache writes exactly
once, before splitting input into components for their respective rates. Output
tokens do not count. For the example, 500000 input tokens use the lower band and
500001 use the long band. Do not add cached-token counters to an input total that
already includes them.

Unknown `reason` is `missing_source`, `unsupported_source`, or
`conflicting_source`. `observed_rules` is a sorted, deduplicated array of objects
with exactly `bucket` (`short_context` or `long_context`), `operator` (`lt`, `lte`,
`gt`, or `gte`), and `input_tokens`. Sort by bucket, operator, then token count.
Missing source evidence uses an empty array. Cutoffs and observed token counts
must be positive integers no greater than 9007199254740991; booleans, strings, and
fractional values are invalid. Extra rule fields are invalid.

Every rule's `provenance` contains exactly a nonblank `locator` of at most 256 UTF-8
bytes and an `evidence_sha256` of 64 lowercase hexadecimal characters. The locator
refers to the root `source_url`; the digest fingerprints normalized public source
evidence. Explicit API conditions in headings, tooltips, and scoped notes establish
cutoffs. Model names, advertised context windows, compaction limits, and subscription
announcements do not establish them.

`default` and `short_context`, when both present in a tier's prices, are aliases
with identical rates and availability. Tier aliases such as `fast` and `priority`
must agree in both prices and rule semantics before a consumer coalesces them.
An unavailable price row is distinct from an unknown applicability rule.

## Compatibility and change detection

Use a consumer that supports schema 2 and timestamp-free model records. Older
Codex Pooler importers reject this output; update the consumer before selecting
this catalog format. Codex Pooler's importer format revision 3 supports it and
also accepts legacy catalogs without `schema_version`.

For compatibility, that importer permits legacy or transitional model timestamps
only when they are valid, nonblank, and equal to root `generated_at`; explicit
null is invalid. A v2 token model must still have its rule. Legacy imports alone
receive an internal `legacy` rule with a tagged 272000-token assumption;
`legacy` is not a valid v2 catalog status or evidence for a new model's cutoff.

The producer's semantic SHA-256 includes schema/rule versions, rates, rule status,
cutoff, and unknown reasons/observations. Generation timestamps and rule provenance
are excluded from semantic comparison. Changing only a cutoff changes semantic
identity even when rates match; changing only a source locator or evidence digest
does not. A byte SHA-256 instead identifies the exact serialized file, including
timestamps, provenance, and formatting.

Publication handles timestamp-layout cleanup separately from semantic equality:
each destination replaces a catalog with repeated model timestamps once, even
when prices are unchanged. Later root-time-only changes do not replace the pricing
object; the S3 manifest can still refresh. The manifest's schema version remains 1
and its pricing hash remains semantic; it is separate from catalog schema 2.

## Codex Pooler

The generated `pricing.json` file is consumed by
[Codex Pooler](https://github.com/icoretech/codex-pooler) for scheduled pricing
snapshot refreshes.

Imported generations are immutable. Revision 3 records the exact root generation
string in `<generated_at>:importer-format-3` and keeps each model/tier's rule and
rates together. Completion selects the applicable generation at completion time,
which can differ from the reservation's generation. Later imports do not reprice
recorded costs or reconstruct missing historical rule facts. Advertised context
windows and compaction remain independent of pricing.
