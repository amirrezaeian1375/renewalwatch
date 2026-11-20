# RenewalWatch

**Category:** Intelligent Contracts
**Network:** GenLayer Studionet

## The problem

Plenty of agreements depend on a domain staying alive. An escrow releases when
the deal closes next year. A licence runs for three more years. A brand purchase
closes only if the name is still owned by the seller. The fact that answers all
of them sits in one public record, the registry record for the domain.

Today someone opens a lookup tool, reads the expiration date, compares it to the
date their contract cares about, and pastes a screenshot into a thread. Nobody
can recompute it a month later, and nobody can see which field of which record
it came from.

RenewalWatch turns that one check into a record on chain: a domain and a cutoff
date go in, and the registry's own expiration event comes back as evidence with
a verdict attached.

## What it does

`open_watch(domain, cutoff)` pins one domain and one cutoff date in ISO form and
returns an id. `resolve(id)` reads the registry record under consensus,
transcribes the calendar day of its expiration event, and derives the verdict by
comparing two ISO strings:

```
OUTLIVES  the registry expiration day is strictly after the cutoff
LAPSES    it is on or before the cutoff
```

ISO dates compare in chronological order as plain strings, so the verdict is a
string comparison inside the contract. No calendar arithmetic, no timezone
maths, no model opinion.

Live, against the .com and .net registries, one record read twice under two
different cutoffs plus one boundary case:

| watch | request | verdict | registry date |
|---|---|---|---|
| `W1` | `google.com`, cutoff `2027-01-01` | `OUTLIVES` | 2028-09-14 |
| `W2` | `google.com`, cutoff `2035-01-01` | `LAPSES` | 2028-09-14 |
| `W3` | `example.net`, cutoff `2027-08-30` | `LAPSES` | 2027-08-30 |

Same record, two cutoffs, two verdicts: the date belongs to the registry and the
question belongs to the caller. `W3` is the boundary, a cutoff equal to the
expiration day, and it lapses because the rule is strictly after.

## Why the consensus layer is load bearing

* **The model transcribes, the contract decides.** The prompt asks for exactly
  one thing: the event object whose action is `expiration`, copied verbatim,
  plus the day part of its date. It does not contain the cutoff and it does not
  ask for an opinion. A test plants a poison mock that only fires if the cutoff
  ever leaks into the prompt, and asserts the honest reading wins.
* **A settled date spends its evidence three times.** Every validator re fetches
  the registry itself and then requires: (1) the quoted event object exists on
  that fresh copy; (2) the object prints the exact date stored beside it; (3)
  the object carries the word `expiration`. A borrowed quote fails, a real
  object carrying a different day fails, and a date lifted from a registration
  or last-changed event fails.
* **The validator then re runs the whole pipeline** and requires an identical
  state, verdict, reason, and date. A disagreement is a no vote, not a stored
  guess. A validator that cannot read the registry at all rejects a settled
  claim rather than certifying what it could not see.
* **A failure cannot become a verdict.** A record with no expiration event is
  stored as `UNRESOLVED` with the reason `NO_EXPIRATION_EVENT`. A non-ISO date,
  an ungrounded quote, and a quote that does not print the stored date each get
  their own reason. A registry that answers with nothing reverts as unreadable.
* **Untrusted record text is passed inside `<source>` tags** with an explicit
  instruction that it is data, never instructions. A test plants an injection
  inside the record and asserts the settled date is still the registry's.

## State

One record per domain and cutoff pair, ids `W1`, `W2`, cap 2000, and opening the
same pair twice returns the same id instead of creating a second record. States
are `PENDING`, `SETTLED`, `UNRESOLVED`. Resolving a settled watch re-reads the
registry and bumps `version`, so a renewal shows up as a moved date and a new
`checked_at` rather than an overwritten history.

Input errors revert before any network use: `DOMAIN_INVALID`, `CUTOFF_INVALID`,
`CAPACITY_REACHED`, `UNKNOWN_CHECK`, `SOURCE_EMPTY_OR_UNREADABLE`. Names outside
the two supported registries are refused at open time rather than guessed at
later.

`resolve` is open to any caller. The registry is public, so anyone may refresh a
record; a caller can trigger a read but cannot pick the outcome.

`rules()` publishes the vocabulary and the limits on chain, including the
evidence rule, so a client can validate a request before writing one and read
the validator's obligation from the contract itself.

## Tests

34 direct mode tests, all passing. `genvm-lint check` clean, 9 methods (2 write,
7 view). They cover idempotent opening, both verdicts and the equal-day
boundary, every unresolved reason, name and cutoff validation, an unreadable and
an oversized registry answer, a validator that must reject a forged date, a
forged verdict, a quote missing from its own fetch, a quote that does not print
the stored date, or a non-ISO date, a validator that cannot read the registry,
a validator facing an unresolved claim its own read settles, a re-resolve that
bumps `version` and picks up a moved date, an injection planted inside the
record, the poison mock that proves the cutoff never reaches the prompt, and
routing between the .com and .net registries.

## Proof on chain

```
contract      0x88F9EC245BEd8f03FAe9553b535e0d128ab2A1D7
deploy tx     0x3c968437cae552c2fca2a87af70d62d602c1eb5fac66f60a0d71cfc456ea4eed  FINALIZED, MAJORITY_AGREE
open W1       0x77792cf82d6c2fbe271c18b4b2b9453831c4491547dd2dcde6516dcb2eeea44c  W1
resolve W1    0x0629360ad119fdeb5d2e7162bb7f7c211f9fefe877b57adecbee5029b3f6139d  SETTLED, OUTLIVES, date 2028-09-14
open W2       0xa143af08d4baa749a5886c203280b9b99f31a7a0ff13c87a24c0ae719a0499c6  W2
resolve W2    0x3fedf58c5c88d6279287fe767873399bddd99a9606dbd2168a9a222fa8348236  SETTLED, LAPSES, date 2028-09-14
open W3       0x457d703f43feec833c50c14de4139efdae07ccf50cd2f1dd162f43f6e3ba9d89  W3
resolve W3    0x027b16953aebbdc82127169534409e59eacc3e6d599d0efe4d41703ddbc60635  SETTLED, LAPSES, date 2027-08-30
```

All seven transactions are FINALIZED with MAJORITY_AGREE. The stored quote on
every record is the registry's own event object, copied verbatim:

```
{"eventAction":"expiration","eventDate":"2028-09-14T04:00:00Z"}
```

At deploy time the contract was opened in Studio from the file in
`contracts/renewalwatch.py`. The contract code carried by the deployment
transaction matches that file byte for byte after line ending normalization,
with the trailing newline dropped by the Studio editor.

## Limits, stated honestly

* .com and .net only, read from the RDAP service of their registry operator.
  Other TLDs are refused at open time rather than guessed at later.
* One expiration event per record. RDAP carries exactly one for these TLDs.
* A stored verdict is a claim about the registry at `checked_at`. Domains
  renew, and re-resolving is how you see that they did.
* A cutoff equal to the registry date lapses. The rule is strictly after, and
  the boundary is demonstrated live by `W3` rather than left to the reader.
* A registry outage or a bot wall reverts as unreadable instead of producing a
  date.
