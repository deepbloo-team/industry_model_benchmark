# industry_model_benchmark

Evaluation data for **industry metric extraction** from public procurement notices.

A model is given the full text of a tender notice (the published notice plus the text of its
attachments) and must return the technical metrics the notice actually specifies: dimensions,
capacities, power ratings, materials, standards, quantities. The reference answer for each
document is a flat JSON object of those metrics.

## Files

| File | Documents | Scope |
| --- | --- | --- |
| `isdm_general.jsonl` | 65 | mixed sectors, one or a few documents per sector |
| `isdm_dechets.jsonl` | 31 | waste collection and treatment |
| `isdm_irve.jsonl` | 13 | EV charging infrastructure (IRVE) |

The two sector files are dense: the same handful of metrics recurs across documents, so they
measure consistency of naming and of units. `isdm_general.jsonl` is sparse (801 distinct field
names over 65 documents) and measures whether a model can name a metric it has never seen before.

## Format

One JSON object per line:

```json
{
  "document_id": "4154c0b9-da4c-49ba-8b4a-83d55646a875",
  "answer": {
    "grammage_rideau_g_m2": 520,
    "hauteur_rideau_cm": [228, 257, 268],
    "marque_tissu_rideau": "gerriets"
  },
  "text": "Réalisation et pose de mobiliers ...",
  "text_length": 49044
}
```

* `document_id` - stable identifier, unique within and across files.
* `answer` - the reference metrics, `{}` when the notice specifies none. 10 of the 109 documents
  have an empty answer on purpose: a model that invents metrics there should be penalised.
* `text` - notice body, then each attachment after a `===== PIÈCE JOINTE : <filename> =====`
  marker. Mostly French. Truncated at roughly 50k characters in `isdm_general.jsonl` and roughly
  35k in the sector files.
* `text_length` - `len(text)`, for convenience.

Personal data is redacted in place with bracketed placeholders (`[NOM]`, `[TEL]`, `[EMAIL]`,
`[SIRET]`, `[ADRESSE]`).

## Answer conventions

Scoring has to compare a predicted key against a reference key, so the references follow fixed
rules:

* Keys are snake_case, at most six words, descriptive without any other context.
* **The unit lives in the key, never in the value**: `grammage_rideau_g_m2: 520`, not
  `grammage_rideau: "520 g/m²"`. A value given in another unit in the source is converted, and
  the key names the unit it was converted to.
* Values are numbers, strings, or lists when the notice gives several values for one metric
  (per model, a range). A few are objects keyed by lot (`{"lot_1": 38.656, "lot_2": 21.131}`)
  when the notice states the metric lot by lot.
* Free-text string values are lowercased and unaccented (`"mousse hr ferme non feu"`). Standards,
  classes and product designations keep their official spelling (`"NF EN 1177"`, `"U4P4"`, `"PU28"`),
  so a scorer should normalise case and accents on both sides rather than compare raw strings.

Two keys that differ only by unit (`contract_duration_months` vs `contract_duration_years`) are
two different metrics, not a match.
