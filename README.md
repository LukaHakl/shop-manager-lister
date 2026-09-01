# shop-manager-lister

Bulk-creates marketplace listings on an admin UI that has no API, by driving a
form described in config rather than in code.

```
$ python -m shopmanager.bulk_creator --csv matched.csv --config config.yaml --dry-run

--dry-run: no browser, nothing saved.

Ancient Stone Golem
  [tab] Info
    creator        select_search #creator-select              = 4821
    title          fill          input[name=title]            = Ancient Stone Golem
  [tab] Pricing
    price          fill          input[name=price]            = 24.00

0 of 20 sampled rows could not be resolved.
Ready for a live run.
```

## The problem

The marketplace's Shop Manager lets you sell physical prints of STLs you have
licensed. Creating a listing there is not a free-text form — it is **bound to a
licensed asset** through two dependent dropdowns: pick a Creator, then pick one
of that creator's 3D objects. Only after both are selected do title,
description, tags and price appear, spread across four tabs.

There is no public API for this surface. It is web-only, so it is Selenium.

Manual creation runs 3–5 minutes per listing across hundreds of products. The
automated path is roughly ten seconds each.

## Approach

**Three scripts, not one.** They stay separate on purpose:

| Stage | What | Why separate |
|---|---|---|
| 1. `library_scraper` | Reads every Creator × 3D-object pair the account owns, out of the form's own dropdowns | Slow (5–10 min on a large library), occasionally needs rerunning |
| 2. `price_matcher` | Joins that against an existing catalogue export to carry across titles and prices | Instant, and gets rerun constantly while tuning the match threshold |
| 3. `bulk_creator` | Drives the form for each matched row | The only one that writes to a live account |

Fusing them would mean re-scraping the entire library every time a threshold
changes by 0.05.

**The form lives in config, not in Python.** Stage 3 is written against an
abstract `tab → field → selector → value` description loaded from YAML:

```yaml
form:
  steps:
    - tab: Info
      field: creator
      selector: "#creator-select"
      action: select_search      # type into the dropdown filter, then pick
      value: "{creator_value}"
      required: true
    - tab: Pricing
      field: price
      selector: "input[name=price]"
      value: "{price}"
```

Two reasons. This marketplace's DOM changes, and the same shape — tabs, fields,
a save — recurs on every other marketplace admin UI, so the driver is reusable
where a hardcoded one is not. It is also what makes stage 3 testable at all:
resolving a plan is pure and covered by tests, and only execution needs a
browser.

## Safety, because stage 3 writes to a live account

**Saves as draft by default. Publishing requires an explicit `--publish`.** The
asymmetry is the point: an unpublished draft is deleted in bulk, a published
listing is not. Every run of a script like this against a live account with a
bad CSV is cleanup work.

- **`--dry-run` walks the plan and reports what it *would* select**, without a
  browser and without touching the account. Use it for every first run against a
  new CSV; it exits non-zero if any sampled row cannot be resolved.
- **`--limit N`** for smoke testing.
- **Checkpoint every 10 products**; a rerun skips completed rows, so a killed run
  resumes without creating duplicates. The checkpoint key is the object value,
  never the row index — a reordered CSV would otherwise resume onto the wrong
  products.
- **Per-row result logging** to `mmf_upload_results_{ts}.csv` with status and
  error, so failures can be re-driven as their own input file.
- **A dropdown returning no match is logged and skipped, never fatal.**

A required field that resolves to empty raises rather than filling the rest — a
half-filled draft looks like a successful run.

## The matching problem

Stage 2 joins on normalised title similarity, and fuzzy title matching produced
false positives **across different creators** in a related project. Two studios
both selling a "Stone Golem" is not a coincidence worth matching on.

Two mitigations, both on by default:

1. **Matches are confined to the same creator** where the source side carries
   creator information. This removes the whole class of error rather than trying
   to score around it. (`--any-creator` turns it off.)
2. **The 20 lowest-scoring matches are printed** at the end for eyeball review.
   Tuning a threshold without seeing the marginal cases is guesswork.

Three output files, and the split is deliberate — the unmatched sets are how
catalogue gaps get found:

```
mmf_matched_products_{ts}.csv     feeds stage 3
mmf_unmatched_library_{ts}.csv    owned assets with no listing  -> stock you are not selling
mmf_unmatched_source_{ts}.csv     listings with no owned asset  -> listings you cannot fulfil
```

## Usage

```bash
pip install -r requirements.txt
python -m pytest                      # 46 tests, no browser needed
```

Start Chrome with the debug port open, once:

```bash
chrome.exe --remote-debugging-port=9222 --user-data-dir="C:\chrome-debug"
```

Then the three stages:

```bash
python -m shopmanager.library_scraper --url https://example.com/products/new
python -m shopmanager.price_matcher --library mmf_library_*.csv --source etsy_export.csv
python -m shopmanager.bulk_creator --csv mmf_matched_products_*.csv --config config.yaml --dry-run
python -m shopmanager.bulk_creator --csv mmf_matched_products_*.csv --config config.yaml --limit 20
```

## Notes and limitations

**Stage 1 captures the internal option value, not just the display text.** Stage
3 selects on the value: display text is localised and occasionally re-worded,
and matching on it alone is fragile. The round-trip — every value stage 1
captures can be selected by stage 3 — is the acceptance criterion.

**Waiting is on option-count stabilisation, not a fixed sleep.** The UI warns the
object list can take up to 30 seconds to populate. A fixed sleep either wastes
time or truncates the list, and a truncated list looks exactly like a creator
who owns fewer objects.

**Provenance.** This ran in production against a real account. That original
source was lost and this repository is a rebuild from the specification those
runs produced, so the approach and every rule here are proven, while the rebuilt
code is verified against its 46 tests rather than re-run live. Use `--dry-run`
and `--limit` on a first run.

**The browser layer is untested by design.** `common/browser.py`, stage 1's
scrape and stage 3's execution need a live Chrome. They are thin precisely so
the plan resolution, the join, the checkpointing and the settling logic are
testable without one.

**`common/` is vendored.** These modules also live in a broader toolkit repo,
duplicated deliberately so this one stands alone.

**The default selectors are placeholders.** They are examples of the shape, not
a working configuration for any particular marketplace — supply your own in
`config.yaml`, which is what the whole design is for.

## Licence

MIT — see [LICENSE](LICENSE).
