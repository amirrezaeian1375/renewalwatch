# RenewalWatch

Ask a domain registry one question and get an auditable answer: will this domain still be registered on the day your agreement depends on?

An escrow releases when a deal closes next year. A licence runs for three more years. A brand purchase closes only if the name is still owned by the seller. The fact that settles all of them lives in one public record, the registry record for the domain, and today someone checks it by hand and pastes a screenshot into a thread. Nobody can recompute it later.

```
open_watch(domain, cutoff)  ->  resolve  ->  OUTLIVES | LAPSES  +  the registry's own expiration event
```

## What it does

`open_watch` pins one domain and one cutoff date in ISO form, such as `2027-01-01`, and returns an id. `resolve` reads the registry record under consensus, transcribes the calendar day of its expiration event, and derives the verdict by comparing two ISO strings:

| Verdict | Meaning |
|---|---|
| `OUTLIVES` | The registry expiration day is strictly after the cutoff. |
| `LAPSES` | It is on or before the cutoff. |

ISO dates compare in chronological order as plain strings, so the verdict is a string comparison inside the contract. No calendar arithmetic, no timezone maths, no model opinion.

A watch is idempotent per pair: opening the same domain and cutoff twice returns the same id instead of creating a second record. Resolving a settled watch re-reads the registry and bumps `version`, so a renewal shows up as a moved date with a new `checked_at` instead of an overwritten history.

## Live proof

Deployed on GenLayer Studio, contract `0x88F9EC245BEd8f03FAe9553b535e0d128ab2A1D7`, deployment transaction `0x3c968437cae552c2fca2a87af70d62d602c1eb5fac66f60a0d71cfc456ea4eed` (FINALIZED, MAJORITY_AGREE).

Three watches against the live .com and .net registries:

| Watch | Request | Verdict | Registry date |
|---|---|---|---|
| `W1` | `google.com`, cutoff `2027-01-01` | `OUTLIVES` | 2028-09-14 |
| `W2` | `google.com`, cutoff `2035-01-01` | `LAPSES` | 2028-09-14 |
| `W3` | `example.net`, cutoff `2027-08-30` | `LAPSES` | 2027-08-30 |

`W1` and `W2` read the same record and disagree on purpose: the date belongs to the registry, the cutoff belongs to the caller. `W3` is the boundary case, a cutoff equal to the expiration day, and it lapses because the rule is strictly after.

Each row is two write transactions, and every one of them finished FINALIZED with MAJORITY_AGREE:

| Step | Transaction |
|---|---|
| deploy | `0x3c968437cae552c2fca2a87af70d62d602c1eb5fac66f60a0d71cfc456ea4eed` |
| open `W1` | `0x77792cf82d6c2fbe271c18b4b2b9453831c4491547dd2dcde6516dcb2eeea44c` |
| resolve `W1` | `0x0629360ad119fdeb5d2e7162bb7f7c211f9fefe877b57adecbee5029b3f6139d` |
| open `W2` | `0xa143af08d4baa749a5886c203280b9b99f31a7a0ff13c87a24c0ae719a0499c6` |
| resolve `W2` | `0x3fedf58c5c88d6279287fe767873399bddd99a9606dbd2168a9a222fa8348236` |
| open `W3` | `0x457d703f43feec833c50c14de4139efdae07ccf50cd2f1dd162f43f6e3ba9d89` |
| resolve `W3` | `0x027b16953aebbdc82127169534409e59eacc3e6d599d0efe4d41703ddbc60635` |

The stored quote on every record is the registry's own event object, copied verbatim:

```
{"eventAction":"expiration","eventDate":"2028-09-14T04:00:00Z"}
```

## Why the verdict is not written by a model

The model is never told the cutoff. Its entire job is to copy one event object out of the record and hand over the day part of its date. The comparison is a string comparison in the contract, so any node reproduces the verdict without consulting a model.

Before a settled answer is stored, an independent validator does four things on its own fetch of the registry:

1. It requires the quoted event object to exist on the record it fetched itself. A hallucinated or borrowed quote fails here.
2. It requires that object to print the stored date. A real object carrying a different day fails here.
3. It requires that object to carry the word `expiration`, so a date lifted from a registration or last-changed event fails here.
4. It re-runs the whole read and requires an identical state, verdict, reason, and date.

Only then is the record written. A record with no expiration event is stored as `UNRESOLVED` with the reason `NO_EXPIRATION_EVENT`, never as a verdict. A registry that answers with nothing reverts as unreadable instead of producing a date.

## API

Writes:

| Method | Arguments | Effect |
|---|---|---|
| `open_watch` | `domain`, `cutoff` | Creates a pending watch, returns its id such as `W1`. Idempotent per pair. |
| `resolve` | `watch_id` | Reads the registry under consensus, stores the verdict, returns the new state. |

Views:

| Method | Returns |
|---|---|
| `get_watch(watch_id)` | The whole record: domain, cutoff, state, verdict, reason, date, quote, `checked_at`, `version`. |
| `verdict(watch_id)` | Verdict string, empty while pending. |
| `expiry_date(watch_id)` | The registry date the record settled on, empty while pending. |
| `outlives(watch_id)` | `true` only for a settled `OUTLIVES` watch. The one call another contract needs. |
| `list_watches()` | Up to 60 records in creation order, each with id, domain, cutoff, state, verdict, and date. |
| `count()` | Number of watches stored. |
| `rules()` | The accepted vocabulary and limits, so a client can validate before writing. |

## Vocabulary and limits

Registries: `com` and `net`, read from the RDAP service of their registry operator. Any other name is refused at open time.

States: `PENDING`, `SETTLED`, `UNRESOLVED`.

Reasons for an unresolved watch: `EXTRACTION_FAILED`, `NO_EXPIRATION_EVENT`, `DATE_NOT_ISO`, `QUOTE_NOT_GROUNDED`, `QUOTE_NOT_SUPPORTING`.

Errors raised by the writes: `DOMAIN_INVALID`, `CUTOFF_INVALID`, `CAPACITY_REACHED`, `UNKNOWN_CHECK`, `SOURCE_EMPTY_OR_UNREADABLE`.

Limits: names up to 100 characters, cutoff and dates as ISO `YYYY-MM-DD` calendar days, registry answers from 32 to 20000 characters, quotes from 12 to 400 characters, up to 2000 watches.

## Try it

Deploy `contracts/renewalwatch.py` in GenLayer Studio, then write:

```
open_watch("google.com", "2027-01-01")
-> "W1"

resolve("W1")
-> "SETTLED"

get_watch("W1")
-> {"id": "W1", "domain": "google.com", "cutoff": "2027-01-01", "state": "SETTLED",
    "verdict": "OUTLIVES", "date": "2028-09-14",
    "quote": "{\"eventAction\":\"expiration\",\"eventDate\":\"2028-09-14T04:00:00Z\"}"}
```

Move the cutoff past the registry date and the same record lapses. Set the cutoff equal to the registry date and it lapses too, because the rule is strictly after.

## Tests

`tests/direct/test_renewalwatch.py` runs 34 tests against the contract with the model and the registry fetch stubbed, covering idempotent opening, both verdicts and the equal-day boundary, every unresolved reason, input validation for names and cutoffs, an unreadable and an oversized registry answer, a validator that must reject a forged date, a forged verdict, a quote missing from its own fetch, a quote that does not print the date, or a non-ISO date, a validator that cannot read the registry and so must not accept a settled claim, a validator facing an unresolved claim its own read settles, a re-resolve that bumps `version` and picks up a moved date, an injection planted inside the record, a poison mock proving the cutoff never reaches the prompt, and routing between the .com and .net registries.

```
pip install -r requirements.txt
pytest tests/direct -q
genvm-lint check contracts/renewalwatch.py
```

## Honest limits

- .com and .net only. Other TLDs are refused at open time rather than guessed at later.
- One expiration event per record. RDAP carries exactly one for these TLDs.
- A stored verdict is a claim about the registry at `checked_at`. Domains renew, and re-resolving is how you see that they did.
- A registry outage or a bot wall reverts as unreadable instead of producing a date.

## Layout

```
contracts/renewalwatch.py           the contract, no dependencies outside the GenLayer runtime
tests/direct/test_renewalwatch.py   direct-mode tests
gltest.config.yaml                  localnet and studionet targets
requirements.txt                    gltest and the linter
```
