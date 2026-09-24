# AAK AMEA target screening pipeline

A five-layer multi-agent screening setup for Claude Code. Layers 1 to 3 produce a normalised, enriched, source-traced candidate pool. Layer 4 scores it deterministically and layer 5 stress-tests the ranking before it reaches AAK.

## Setup
1. Unzip into a folder and open it in Claude Code.
2. Read `CLAUDE.md`. It tells the main session how to run each layer.
3. Check `criteria/screening-criteria.md` section 13 and update anything AAK has since confirmed.

## Layout
```
CLAUDE.md                        orchestration instructions, read automatically by Claude Code
criteria/screening-criteria.md   single source of truth for scope and criteria
criteria/scoring-weights.json    every layer 4 tunable, AAK's to edit
docs/discovery-rules.md          shared rules for layer 1 agents
docs/enrichment-rules.md         shared rules for layer 3 agents
docs/scoring-rules.md            layer 4 methodology: dimension -> sub-metric -> rule -> field
docs/review-rules.md             layer 5 methodology, the four tests plus the portfolio check
schemas/                         output shapes for every layer
scripts/merge_raw.py             layer 2, mechanical merge, dedupe and exclusion pass
scripts/layer2_rules.py          layer 2 shared classifier: exclusions, bucket_hint, flags
scripts/validate.py              layer 3 gate + `--candidates` layer 2 gate
scripts/score.py                 layer 4, deterministic scoring over the enriched files
.claude/agents/                  eleven subagents
data/raw/                        layer 1 output, one jsonl per agent per run
data/candidates.csv              layer 2 output (+ bucket_hint, excluded_activity_flag, availability_note)
data/needs_rediscovery.csv       rows that could not be screened; input to a second discovery pass
data/enriched/                   layer 3 output, one json per candidate
data/exclusions.csv              persistent reject list (with criteria_section)
criteria/trade-agreements.md     country-pair / bloc FTA matrix for criteria 8.3 (DRAFT)
data/gate-judgements.csv         human verdicts, layer 4 mandatory criteria that can't be sourced per candidate
data/overrides.csv               human promote/demote/exclude calls, never touches the score
data/scored.csv, scorecards/, longlist.md   layer 4 output: ranking, audit trail, deliverable
data/review/                     layer 5 output, one file per reviewed candidate plus a portfolio check
data/runlog.md                   methodology trail
```

## Agents
| Layer | Agent | Source universe |
|---|---|---|
| 1 | discover-registry | Association member lists, licensing, certification databases |
| 1 | discover-tradeshow | Exhibitor lists, Fi Asia to in-cosmetics |
| 1 | discover-tradedata | HS code flows, customs, port and export body lists |
| 1 | discover-dealflow | Deal press, PE portfolios, exchange announcements |
| 1 | discover-customer-backtrack | FMCG palm and fats supplier disclosures |
| 1 | discover-frontier-tech | Fermentation, cellular, algal, Power-to-X |
| 2 | normalize | Entity resolution, parent rollup, status, hard exclusions |
| 3 | enrich-ownership | Legal entity, parent, shareholders, deal history |
| 3 | enrich-operations | Sites, value chain, products, quadrants, certs, customers |
| 3 | enrich-financials | Revenue, EBITDA, volumes, EBIT per kg proxy, EV range |
| 4 | *(script only)* | `scripts/score.py`: dimension scores, gates, tiers, quadrants, sensitivity |
| 5 | adversarial-review | Hallucination check, thin-data check, missed exclusions, rank challenge, portfolio concentration |

## Run order
```
# layer 1, all six in parallel
> Use the discover-registry, discover-tradeshow, discover-tradedata, discover-dealflow,
  discover-customer-backtrack and discover-frontier-tech subagents in parallel.

# layer 2
$ python scripts/merge_raw.py
> Use the normalize subagent on data/candidates_draft.csv.
$ python scripts/validate.py --candidates

# layer 3, by bucket_hint (see CLAUDE.md), batches of 10 to 15 within a bucket
> Use enrich-ownership, enrich-operations and enrich-financials on C0xx to C0yy.
$ python scripts/validate.py C0xx ... C0yy

# layer 4
$ python scripts/score.py --init-gates          # once, then fill in data/gate-judgements.csv
$ python scripts/score.py --as-of 2026-09-04 --scenarios --report

# layer 5
> Use the adversarial-review subagent on the default batch, or a specific ID list.
$ python scripts/score.py --apply-review        # after review, see the score impact
```

## Design choices worth knowing
- No company enters any file without a URL where its name appears. This is the only defence against hallucinated targets and it is enforced by both scripts.
- Discovery is split by source type, not geography, because a geography agent will return the ten companies everyone knows and then invent an eleventh.
- Scoring runs as code over the enriched files, not by judgement during discovery or enrichment, so the ranking can be rerun and defended. `scripts/score.py` holds no thresholds or judgements of its own: every band, lookup, keyword list and gate definition lives in `criteria/scoring-weights.json`, which is AAK's to edit.
- Nulls are correct outputs, and they say why. Every null field carries a `null_reason` (`no_public_data`, `login_walled`, `conflicting_sources`, `not_applicable`), enforced by `scripts/validate.py`. A missing value is never scored as average: layer 4 drops an unknown sub-metric from the calculation instead of imputing a neutral score, so a candidate with no data cannot outrank one that was actually assessed. `data_completeness` and the `quadrant` column carry the difference between "scored low" and "not yet known" all the way to the deliverable.
- Three mandatory criteria (potential for global leadership, favourable growth, ability to differentiate) and the market-size floor cannot be established from one company's enriched record. They are recorded as human verdicts in `data/gate-judgements.csv`, not approximated in code; an unfilled verdict resolves to `review`, never to a quiet `pass`.
- Layer 5 cannot change a score. Its findings reach one only through `scripts/score.py --apply-review`, run by a person, and a human override in `data/overrides.csv` never touches `total_score` or `rank`. What AAK sees is the evidence-based score, the red team's verdict, and the human decision, side by side.

## Outstanding
The pipeline is complete end to end. What remains is AAK's own input: confirming the weights and lookups in `criteria/scoring-weights.json`, filling in `data/gate-judgements.csv`, and resolving the open questions in `criteria/screening-criteria.md` section 13 (the size ceiling's currency and basis, the EBIT-per-kg baseline, the definition of a stable region, whether the frontier-tech track runs on separate criteria, and long-list size among them). Every one of those is called out in `data/longlist.md` each time it is generated.
# Acquisition-targets
