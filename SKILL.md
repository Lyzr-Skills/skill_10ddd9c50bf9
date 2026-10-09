---
name: bucket-parties-geography
description: Fact bucket for WHO is in the transaction and WHERE - customer and Anaplan contracting entities, resellers, end customers, partners, affiliates and other named organisations, with every printed address (bill-to, ship-to, sold-to, registered office, notice, end-user location) split into parts with country and state / province, plus tax and registration IDs. Feeds customer-name matching, region, territory screening, contracting-entity and tax-code checks, government and reseller flags.
metadata:
  bucket_id: parties_geography
  schema_version: 2.0.0
  output_model: parties_geography_models.py (PartiesGeographyOutput)
  brd_refs: Sec 3.1, 3.3, 3.4, 6.1, 7.1, 7.2, 8.4
  tracker_rows: 8, 19, 20, 47, 48, 57
---

# Bucket: Parties & Geography

This skill is self-contained: Part A is how to extract (shared by every bucket), Part B is what
this bucket extracts, Part C is the exact output structure.

# Part A - How to extract

You extract facts for **one bucket** (Parties & Geography) from **one document**. Part B says which facts,
Part C gives the exact output structure, and the output is validated by `parties_geography_models.py`
(`PartiesGeographyOutput`). Your output feeds three kinds of downstream checks:

| Output section | Downstream consumer | Your job |
|---|---|---|
| `values` | Fact rules in code (exact match, tolerance, date difference) | Transcribe the single value exactly + light parse |
| `records` | Calculation rules in code (sums, joins, period comparisons) | Transcribe every row of a repeating structure, completely, with stable keys |
| `evidence` | Contextual agent (policy overrides, deviations, judgements) | Capture verbatim clauses and signals; give non-binding hints only |

**Golden rule: you transcribe; code and the contextual agent decide.**

## A1. Input
- An OCR reconstruction of the PDF in **markdown**. Pages are separated by page markers
  `<<<PAGE n>>>`, where `n` is the 1-based physical page of the PDF.
- Markdown formatting in the input (`#` headings, `**bold**`, `_italic_`, `|` table cells, `-` bullets,
  `>` quotes, backticks, `<br>`, `\` escapes) is layout produced by the OCR step, not document text.

Use nothing else: not Salesforce, not Anaplan, not other documents, not general knowledge about the
customer. If a fact is not in this document, it is `not_found`.

## A2. Applicability
Read Part B section 1 and the document.
- Bucket does not apply -> `applicable: false`, a one-sentence `applicability_reason`, and
  `values`, `records`, `evidence` set to `null`. `observations` and `notes` are `[]`.
- Bucket applies -> `applicable: true` and fill **every** field Part B defines.

"Applies" means the document is the kind that could contain these facts, not that it does.
Silence in an applicable document is evidence (no opt-out clause, no budget cap), so it is reported
as `not_found`, never skipped.

## A3. Citations
Every `found`, `inferred` or `unreadable` fact carries `sources`, a list of
`{"page_number": int, "quote": [string, ...]}`. The quotes are used to **highlight the fact in the
original PDF**, so they must match the OCR text exactly.

**Rules for each string in `quote`:**
1. It is a verbatim span from **one single line** of the page. Never join two lines in one string.
2. It contains **plain text only**: no markdown syntax (`#`, `**`, `__`, backticks, `>`, leading
   `-` / `*` / `+` bullets, `<br>`, HTML tags, `\` escapes) and **never a `|` character**. Drop the
   formatting characters and keep the words between them: `**Order Schedule Number:** ACM-1`
   is quoted as `Order Schedule Number: ACM-1`.
3. Copy the characters exactly as the OCR text has them, including OCR typos, casing, punctuation,
   currency symbols and number formatting. No ellipses, no paraphrase, no translation.
4. Make it long enough to identify the fact unambiguously on the page (label + value, or the
   sentence that states it), but no longer than the line.

**When a fact spans several lines or cells, use one string per line or cell, in reading order:**
- Address block on 5 lines -> 5 strings.
- Label on one line, value on the next -> 2 strings.
- Table row -> one string per relevant cell (`| Solution Architect | 200 |` ->
  `["Solution Architect", "200"]`).
- A printed value that itself contains `|` (e.g. DocuSign `10-Apr-26 | 5:35:23 PM EDT`) -> split at
  the pipe: `["10-Apr-26", "5:35:23 PM EDT"]`. The `raw` field still keeps the full printed value.
- A fact on two pages -> two source objects, one per page.

**By status:**
- `inferred`: quote the text you inferred from. Never invent text for something that is absent.
- `unreadable`: quote the partially legible text as the OCR gives it; if nothing is legible, use
  `["[illegible]"]` with the page number.
- `not_found` / `not_applicable`: `sources` is `[]`.

## A4. Status and confidence
| Status | Use when | raw / parsed |
|---|---|---|
| `found` | Stated in the document | raw = as printed (formatting removed); parsed = light parse |
| `inferred` | Not stated directly but follows from specific text | raw = the text relied on; parsed = your reading; confidence at most `medium` |
| `not_found` | Bucket applies, document is silent | both null, no sources, no candidates |
| `not_applicable` | This field cannot exist in this document type | both null, no sources, no candidates |
| `unreadable` | Text is there but illegible | raw = what is legible; parsed = null |

`confidence`: `high` = explicit and unambiguous; `medium` = explicit but wording or formatting
leaves some doubt, or inferred; `low` = conflicting or barely legible.

## A5. Light parsing
`raw` is the truth (as printed, markdown removed); `parsed` is a convenience that code re-checks.
Put `parsed: null` rather than guessing.

| Type in Part C | parsed format | Rules |
|---|---|---|
| Text | string | Trim whitespace and line breaks; do not rewrite. |
| Number | number | Remove thousands separators; respect decimal commas (`1.234,56` -> 1234.56). Percentages in percent units (`99.5%` -> 99.5). |
| Boolean | true / false | Only from explicit wording; otherwise null. |
| Date | `YYYY-MM-DD` | Only if unambiguous. `03/04/2026` with no locale evidence -> null + `normalization` note. |
| Time | `HH:MM:SS` (24 h) | Seconds `00` if not printed. |
| Timestamp | `YYYY-MM-DDTHH:MM:SS` | Local time as printed; **no conversion, no offset**. |
| UtcOffset | `+HH:MM` / `-HH:MM` | Only when printed as an offset or an unambiguous zone listed in Part B. |
| Money | `{"amount": number, "currency_iso": "USD"}` | ISO 4217. `$` alone is ambiguous -> `currency_iso: null` unless the document names the currency. |
| CountryCode | ISO 3166-1 alpha-2 (`US`, `GB`) | |
| SubdivisionCode | ISO 3166-2 (`US-IL`, `CA-ON`) | |
| LanguageCode | ISO 639-1 (`en`, `fr`) | |
| enum (`"a" \| "b"`) | one listed value | If nothing fits, use `other` where listed and keep the wording in raw. Never force a fit. |

**Never compute.** No totals, sums, differences, term lengths, execution dates, discounts, FX or
time-zone conversions. If the document prints a total, transcribe it. Code does the arithmetic.

## A6. Conflicts -> candidates
If the document states the same fact differently in two places, put the reading you judge primary
in the value, every other reading in `candidates` (each with its own sources), set
`confidence: low`, and add a `conflict` note. Do not reconcile.

## A7. Records (repeating structures)
- One item per row / entry **as printed**. Do not merge, split, de-duplicate or re-order.
- `key`: follow the key rule in Part B. Keys must be unique within the collection.
- Leaves are `{"raw", "parsed"}`; `status` and `sources` sit on the item (quote the row's line or cells).
- A column that is blank for a row -> `{"raw": null, "parsed": null}`.
- `completeness`: `items_extracted` = number of items you output; `items_declared` = a count the
  document itself prints, else null; `continues_beyond_extraction` = true if the table visibly
  continues somewhere not in this document; `status` = `complete` / `partial` / `unknown`.
- `declared_totals`: every total / subtotal printed for the collection, transcribed.
- Empty collection: `items: []`, `items_extracted: 0`, status `complete`.

## A8. Evidence (for the contextual agent)
**Clauses**: capture each clause matching one of Part B's clause categories.
- `clause_id`: `C1`, `C2`, … in document order.
- `text_verbatim`: the clause text with markdown formatting removed and line breaks kept.
- Length: full text for categories marked ★ (full-verbatim). Otherwise up to 1,500 characters; if
  cut, set `text_truncated: true` and stop at a sentence end.
- `sources`: one string per line of the clause (and one source object per page).
- `overrides_hint`: fill it when the clause says it modifies, replaces, supplements or conflicts with
  another document or clause; otherwise null.

**Signals**: short factual pointers explaining why something may matter.
- `signal_id`: `S1`, `S2`, … Link `clause_id` when the signal comes from a captured clause.
- Record weak signals too. `description` states what is printed, not a conclusion.

**Hints**: one entry for every hint Part B defines, every time the bucket applies.
- `likely` must cite at least one `signal_id`; `unlikely` / `unclear` may have an empty list.
- Hints are leanings, not verdicts.

## A9. Observations and notes
- `observations`: useful facts this bucket does not define but that matter to contract review
  (cited). Do not repeat facts already in a field.
- `notes`: `conflict`, `normalization`, `missing` (expected but absent), `multi_document`,
  `truncation`, `other`, with `field_path` when about a specific field.

## A10. Fact paths
Downstream checks, agent findings and human overrides address facts by path:
`parties_geography.values.<field>`, `parties_geography.records.<collection>[<key>].<leaf>`,
`parties_geography.evidence.clauses[<clause_id>]`, `parties_geography.evidence.signals[<signal_id>]`.

## A11. What the validator checks
`parties_geography_models.py` rejects the output if any of these fail; it is retried once with the errors,
then routed to a human.
1. Structure, field names, types and enum values match Part C exactly (no extra fields).
2. Every quote string is plain text (no `|`, no markdown syntax, no line break) and is found on a
   single line or table cell of its page.
3. Status rules: `found` / `inferred` / `unreadable` have sources; `not_found` / `not_applicable` have
   none and null raw / parsed; `inferred` is never `high` confidence.
4. `values` / `records` / `evidence` are null exactly when `applicable` is false.
5. `items_extracted` equals the number of items; record keys are unique; every item has a source.
6. Clause and signal IDs are unique and every reference to them exists.
7. Every hint appears exactly once; `likely` hints cite a signal; ★ clauses are not truncated.

# Part B - Parties & Geography

## 1. Applicability
**Applies** to any document that names an organisation as a party, or prints an address for one:
OS, SSA / customer paper, SOW, change order, amendment, PO, partner agreement, marketplace private
offer, quote.

**Does not apply** to documents with no named organisations or addresses, e.g. a generic SLA
exhibit or DPA incorporated by reference, or an internal email that only names individuals.

## 2. Ownership boundaries
| Fact | Owner bucket | Note |
|---|---|---|
| Signer names and titles | Execution & Signatures | individuals are never `parties` here |
| Marketplace channel (Azure / AWS) | Document Identity & References | the marketplace operator as an organisation may still appear here with role `marketplace_operator` |
| Partner performing services (as written in the SOW) | Services Delivery (`delivering_partner_raw`) | the partner's legal entity and address are here |
| Non-appropriation / fiscal-funding clause text | Term & Termination | here only as a `public_sector_terms` signal |
| Governing law and courts | Legal Protections | - |
| Tax code, tax rate, withholding | Billing, Payment & Tax | tax / VAT registration IDs printed with a party are here |

## 3. values (fact rules)
| Field | Type | Extraction notes |
|---|---|---|
| `customer_legal_name` | Text | Customer contracting party as printed in the preamble / party block; if none, the signature block. If the name is printed differently elsewhere (signature block, bill-to, footer), put the other spellings in `candidates`, keep `confidence` at `medium` or lower, and add a `name_variant` signal. |
| `anaplan_contracting_entity` | Text | The Anaplan party exactly as printed. Copy suffixes exactly (`Anaplan, Inc.`, `Anaplan Limited`, `Anaplan (UK) Limited`): the entity checks depend on the exact name. A different Anaplan name in the signature block -> `candidates` + `entity_name_mismatch` signal. |

Both values are the primary pick; the full list of organisations is in `records.parties`.

## 4. records (calculation and matching rules)

### parties
| Leaf | Type | Notes |
|---|---|---|
| `role` | enum `customer`, `anaplan_entity`, `end_customer`, `reseller`, `distributor`, `partner_subcontractor`, `affiliate`, `parent_company`, `marketplace_operator`, `guarantor`, `beneficiary`, `other`, `unclear` | Use the role the text gives. When the text names an organisation without making its role clear, use `unclear` and let the agent decide from the signals. |
| `legal_name` | Text | As printed. |
| `defined_term` | Text | The defined term in quotes, e.g. "Customer", "Anaplan", "Reseller". |
| `entity_form` | Text | Wording describing the entity: "a Delaware corporation", "an agency of the State of Victoria", "as represented by the Minister of Finance". |
| `registration_number` | Text | Company / registration number. |
| `tax_id` | Text | VAT / GST / HST / EIN printed for the party (not with a specific address). |
| `is_contracting_party` | Boolean | true only for organisations that are party to this document. |

Key: `<role>:<n>` counting per role in document order (`customer:1`, `anaplan_entity:1`, `reseller:1`).

### addresses
| Leaf | Type | Notes |
|---|---|---|
| `address_type` | enum `bill_to`, `ship_to`, `sold_to`, `registered_office`, `principal_place_of_business`, `notice`, `remit_to`, `end_user_location`, `party_address`, `other` | From the label printed on the block. An address printed with a party but no label -> `party_address`. |
| `entity_name` | Text | Organisation named on the address block (may differ from the party name, e.g. a ministry under a government party). |
| `raw_block` | Text | The whole block exactly as printed; parsed = single line. Always filled, so nothing is lost if parsing is imperfect. |
| `address_lines` | Text[] | Street lines only (no city, region, postcode, country). |
| `city` | Text | |
| `subdivision` | Text | State / province / county. parsed = ISO 3166-2 code when clear (`US-IL`, `CA-ON`, `IN-MH`), else null. Needed for Canada and India tax codes (BRD 3.4). |
| `postal_code` | Text | |
| `country` | Text | raw = as printed; parsed = ISO 3166-1 alpha-2 (`US`, `GB`, `AE`, `SA`, `CN`, `HK`). If the country is not printed, raw null and parsed may be filled only when the city / postcode / subdivision makes it unambiguous (e.g. `Toronto, ON` -> `CA`); add a `normalization` note. |
| `contact_name`, `contact_email` | Text | Billing / notice contact printed in the block. |
| `tax_id` | Text | Tax ID printed inside this address block. |

Key: `<party key>/<address_type>`, adding `:<n>` for repeats (`customer:1/bill_to`, `customer:1/ship_to:2`).

Rules:
- Every address must link to a party; if the organisation is not otherwise named, add it to
  `parties` (role `unclear` if needed).
- **Extract only printed addresses.** Never copy bill-to into ship-to or infer a missing one. If an
  expected block is missing (e.g. no ship-to on an OS), add a `missing` note. Code applies the
  fallback (ship-to -> bill-to -> sold-to -> registered office) and records that it did.
- Capture every address, including notice and Anaplan addresses: territory screening covers all of them.

## 5. evidence (contextual agent)
**Clause categories** (★ = full-verbatim):
`party_definition`, `affiliate_rights`, `reseller_arrangement` ★, `end_user_terms` ★,
`public_sector_terms` ★ (procurement law, FAR / DFARS, government-specific terms),
`export_sanctions` ★, `usage_location` (where users / data may be located), `other_parties`.

**Signal types**:
| Type | Record when |
|---|---|
| `public_sector_name` | A party name contains Government, Ministry, Department, Agency, State of, Province of, City / County of, Federal, Public Authority, Crown, "His / Her Majesty", university / hospital authority |
| `public_sector_terms` | Procurement law, FAR / DFARS, non-appropriation or fiscal funding, freedom-of-information, sovereign immunity wording |
| `reseller_language` | Order placed through, invoiced by or resold by another organisation |
| `end_customer_named` | An end user / end customer different from the contracting customer is named |
| `affiliate_named` | Affiliates named as users, buyers or beneficiaries |
| `name_variant` | The same party is named differently in different places |
| `entity_name_mismatch` | The Anaplan entity differs between places, or a regional Anaplan name appears on a subscription document |
| `restricted_territory_mention` | Any mention of Cuba, Iran, North Korea, Syria, Belarus, Russia, China / PRC (mainland), Democratic Republic of Congo, or Crimea / occupied regions, in any address or term |
| `multiple_countries` | Parties or addresses span more than one country |
| `other` | Any other party or geography point a reviewer should see |

**Hints** (each exactly once):
| Hint | Lean on… | Feeds |
|---|---|---|
| `public_sector` | Customer is a government / public-sector body | #47 Government |
| `reseller_deal` | A reseller, distributor or marketplace stands between Anaplan and the customer | #48 Reseller |
| `end_customer_differs` | The organisation that uses the service differs from the contracting customer (group procurement affiliates buying for their own group do NOT count) | #48 Reseller |

## 6. Code-side derivations (never done by the model)
- **Primary bill-to / ship-to country**: customer `bill_to` and `ship_to` addresses; fallback order
  ship-to -> bill-to -> sold-to -> registered office, recording which address was used.
- **Region**: country -> AMER / EMEA / APAC from the BRD 3.3 playbook table.
- **Territory category**: country -> Category 1 / 2 / 3 from the playbook (checked on every address).
- **Expected entities**: country -> subscription entity (SSA 12.3) and services entity (playbook).
- **Tax code**: ship-to country (+ subdivision for Canada and India) -> BRD 3.4 table.

## 7. Validations fed (rule registry)
| Rule | Type | Check | Fact paths |
|---|---|---|---|
| #8 Customer | fact | Normalised fuzzy match vs SF `Account.Name` (>= 90% MATCH, 70-90% PARTIAL, < 70% MISMATCH); before flagging, check candidates and the SF account hierarchy | `values.customer_legal_name` (+ candidates) |
| #19 / #20 Region, Sub-region | fact | Derived region vs SF `Sales_Region__c`; sub-region country consistency | `records.addresses[customer:1/bill_to \| ship_to].country` |
| BRD 3.3 Category 3 | fact | Any address in a prohibited country -> critical blocking exception; China bill-to / ship-to -> unbookable, roll back to Stage 4; DRC -> SDN screening | `records.addresses[*].country`, signal `restricted_territory_mention` |
| BRD 3.3 Category 2 | fact | APAC / EMEA country not in the approved list -> Finance approval required | customer address countries |
| BRD 3.3 Subscription entity | fact | OS / SSA Anaplan party = `Anaplan, Inc.` (AMER) or `Anaplan Limited` (EMEA / APAC) for the customer's country; a regional name on a subscription document = exception | `values.anaplan_contracting_entity` + derived region |
| BRD 7.2 s1 SOW entity | fact | SOW Anaplan party = local services entity for the customer's country (playbook) | `values.anaplan_contracting_entity` on the SOW |
| BRD 7.2 s2 Address match | calc | SOW customer address = PO bill-to address (normalised parts; raw blocks shown to the reviewer) | `records.addresses` on SOW and PO |
| BRD 3.4 Tax code | fact | Recommended tax code from ship-to country / subdivision and document type (subscription vs services) | derived ship-to country + `subdivision` |
| BRD 8.4.1 / #57 Cash hold | fact | Ship-to country `SA` or `AE` -> "Commission Hold - Cash Receipt Required" | derived ship-to country |
| BRD 7.1 Partner entity | fact | Partner on the partner agreement = partner named in the SOW | `records.parties[partner_subcontractor:*]`, Services Delivery `delivering_partner_raw` |
| #47 Government | context | Agent decides; open point: client definition (sample tracker shows Government of Ontario = FALSE) | hint `public_sector`, signals `public_sector_*`, `parties[*].entity_form` |
| #48 Reseller | context | Agent decides reseller / end customer | hints `reseller_deal`, `end_customer_differs`, `parties[reseller, end_customer, distributor]` |
| BRD 12 PO correction | fact | PO address mismatch -> route to Order Management for a corrected PO | address match result |

# Part C - Output structure

Return **one JSON object** exactly in this shape. Every key shown is required (use `null` /
`[]` as the rules allow); no other keys are allowed.

## C1. Shared shapes

```jsonc
Source        = { "page_number": int >= 1, "quote": [string, ...] }        // >= 1 plain single-line string
Value<T>      = { "raw": string | null, "parsed": T | null,
                  "status": "found" | "not_found" | "not_applicable" | "inferred" | "unreadable",
                  "confidence": "high" | "medium" | "low",
                  "sources": [Source], "candidates": [Candidate] }
Candidate     = { "raw": string, "parsed": string | number | boolean | Money | null, "sources": [Source] }
Leaf<T>       = { "raw": string | null, "parsed": T | null }
Money         = { "amount": number | null, "currency_iso": string | null }   // ISO 4217, 3 letters
Item          = { "key": string, "status": "found" | "inferred" | "unreadable", "sources": [Source] (>= 1), ...leaves }
Collection<I> = { "completeness": Completeness, "declared_totals": [DeclaredTotal], "items": [I] }
Completeness  = { "items_extracted": int, "items_declared": int | null,
                  "continues_beyond_extraction": boolean, "status": "complete" | "partial" | "unknown" }
DeclaredTotal = { "label_raw": string, "value": Leaf<Money>, "sources": [Source] }
Clause        = { "clause_id": string, "clause_number_raw": string | null, "heading_raw": string | null,
                  "text_verbatim": string, "text_truncated": boolean, "category_hints": [category],
                  "overrides_hint": OverridesHint | null, "sources": [Source] }
OverridesHint = { "target_document_raw": string | null, "target_clause_raw": string | null,
                  "relationship": "modifies" | "replaces" | "supplements" | "conflicts" | "unclear" }
Signal        = { "signal_id": string, "signal_type": signal_type, "description": string, "entity_raw": string | null,
                  "strength": "strong" | "moderate" | "weak", "clause_id": string | null, "sources": [Source] }
Hint          = { "hint": hint_name, "value": "likely" | "unlikely" | "unclear", "signal_ids": [string] }
Observation   = { "label": string, "value_raw": string, "sources": [Source] }
Note          = { "type": "conflict" | "normalization" | "missing" | "multi_document" | "truncation" | "other",
                  "field_path": string | null, "message": string }

Scalar types: Text = string, Number = number, Boolean = boolean, Date = "YYYY-MM-DD",
Time = "HH:MM:SS", Timestamp = "YYYY-MM-DDTHH:MM:SS", UtcOffset = "+HH:MM" | "-HH:MM",
CountryCode = "US", SubdivisionCode = "CA-ON", LanguageCode = "en", Money as above.
```

## C2. `parties_geography` output

```jsonc
{
  "bucket": "parties_geography",
  "schema_version": "2.0.0",
  "applicable": true,                // false -> values, records, evidence are null
  "applicability_reason": string,

  "values": {
    "customer_legal_name": Value<Text>,
    "anaplan_contracting_entity": Value<Text>
  },

  "records": {
    "parties": Collection<{
      "key": string, "status": ItemStatus, "sources": [Source],
      "role": Leaf<"customer" | "anaplan_entity" | "end_customer" | "reseller" | "distributor" | "partner_subcontractor" | "affiliate" | "parent_company" | "marketplace_operator" | "guarantor" | "beneficiary" | "other" | "unclear">,
      "legal_name": Leaf<Text>,
      "defined_term": Leaf<Text>,
      "entity_form": Leaf<Text>,
      "registration_number": Leaf<Text>,
      "tax_id": Leaf<Text>,
      "is_contracting_party": Leaf<Boolean>
    }>,
    "addresses": Collection<{
      "key": string, "status": ItemStatus, "sources": [Source],
      "address_type": Leaf<"bill_to" | "ship_to" | "sold_to" | "registered_office" | "principal_place_of_business" | "notice" | "remit_to" | "end_user_location" | "party_address" | "other">,
      "entity_name": Leaf<Text>,
      "raw_block": Leaf<Text>,
      "address_lines": [Leaf<Text>],
      "city": Leaf<Text>,
      "subdivision": Leaf<SubdivisionCode>,
      "postal_code": Leaf<Text>,
      "country": Leaf<CountryCode>,
      "contact_name": Leaf<Text>,
      "contact_email": Leaf<Text>,
      "tax_id": Leaf<Text>
    }>
  },

  "evidence": {
    "clauses": [Clause],
    "signals": [Signal],
    "hints": [Hint]
  },

  "observations": [Observation],
  "notes": [Note]
}
```

Allowed lists for this bucket:

- `category` (clause `category_hints`): `party_definition`, `affiliate_rights`, `reseller_arrangement` ★, `end_user_terms` ★, `public_sector_terms` ★, `export_sanctions` ★, `usage_location`, `other_parties`
  (★ = full-verbatim, never truncated)
- `signal_type`: `public_sector_name`, `public_sector_terms`, `reseller_language`, `end_customer_named`, `affiliate_named`, `name_variant`, `entity_name_mismatch`, `restricted_territory_mention`, `multiple_countries`, `other`
- `hint_name` (each exactly once): `public_sector`, `reseller_deal`, `end_customer_differs`
