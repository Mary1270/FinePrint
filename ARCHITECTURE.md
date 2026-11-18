# FinePrint architecture

## Entities

- **Prefix**: an https URL prefix with a label, added by the contract owner. Versions can only be registered under a current prefix. When a URL matches more than one prefix, the longest (most specific) one is used.
- **Evidence binding**: at registration the matched prefix and its label are copied onto the version (`prefix`, `label`). This binding is immutable and is the version's authority for both challenges and comparisons: counter evidence must sit under the stored `prefix`, and two versions can only be compared when their stored prefixes are equal. The owner's current prefix list is never consulted for an existing version.
- **Version**: id `v1`, `v2`, ... with the canonical page URL, registrant, state, optional assurance, rating, and challenge data. One URL maps to one version; a URL whose version ended UNRATED may be registered again under a new id.
- **Rating**: level, score, per-field points, the list of rule hits (`why`), the derived fields, and the grounded quotes (`raw`) they came from.
- **Comparison**: one stored record per ordered pair `old>new`, holding the per-field changes with BETTER / WORSE / SAME marks.

## Version state machine

```
register_terms -> REGISTERED
REGISTERED --assess RATED--> RATED
REGISTERED --assess UNDETERMINED (attempts+1)--> REGISTERED, or UNRATED on the 3rd attempt (assurance refunded)
REGISTERED --stale, expire_version--> UNRATED (assurance refunded)
RATED --challenge_assessment, before challenge_end--> CHALLENGED (tries+1)
RATED --finalize after challenge_end--> FINAL (assurance refunded)
CHALLENGED --resolve, merged level differs--> FINAL (challenger: bond + assurance), rating replaced, original_level kept
CHALLENGED --resolve, merged level equal--> FINAL (registrant: bond + assurance)
CHALLENGED --resolve, counter page lacks the version URL--> FINAL (registrant: bond + assurance)
CHALLENGED --resolve undetermined / stale cancel_challenge--> RATED, bond refunded,
      challenge_end = now + window while tries < 3, otherwise challenge_end = now
```

Every state has a path to a terminal state (FINAL or UNRATED) that needs no particular party. A consensus failure on the network leaves storage untouched, and the stale timers (`stale_after_seconds`) then open `expire_version` or `cancel_challenge`. A bounce reopens the window for at most two extra windows in total, because the third filing closes it. Only FINAL ratings are served by the registry views and used by `compare`.

## Extraction and rating pipeline

`_assess_single` (and, for the counter page, `_assess_challenge`) runs identically on the leader and on every validator:

1. Fetch the page, truncated to 15000 characters. An empty page is UNDETERMINED.
2. One model call returns, for five fields, a flag and an exact quote. The prompt tells the model not to judge or rate.
3. `_ground` rebuilds the answer: a flag survives only when it is one of the field's allowed values, its quote is 6 to 300 characters long, the normalised quote is a substring of the normalised page, and the quote contains a topical keyword for that field. Extra rules: a NO answer for sharing or auto-renewal and a NONE answer for refunds need a negative word in their quote; a YES or NONE answer is dropped when its quote denies the fact (for example "we do not share", "we do not require arbitration", "do not renew", or a refund period) and has no contrast word such as "but" or "except"; and INDEFINITE needs an indefinite-retention phrase without a nearby negation. Otherwise the field becomes UNSTATED with an empty quote. A model value of `None` or any non-string becomes an empty string, never the text "None".
4. `_derive` parses periods from the quotes (digits, number words and "a"/"an" before a unit; the longest duration in the quote) and buckets them. `_rate` adds up the points and assigns the level. Fewer than two non-UNSTATED fields is UNDETERMINED.

For a challenge, the counter page must first quote the full version URL as a standalone token (no extra characters after it, no `.bak`, no query string, no different path; the host matches in any case, the path is case-sensitive). Only then is it extracted. Its stated fields replace the same fields in the stored `raw`, and the merged result is re-scored. The main page is not fetched again.

## Consensus

`gl.vm.run_nondet_unsafe(leader, validator)` is called with two positional arguments from module-level functions (`_consensus_single`, `_consensus_challenge`) whose closures capture only plain strings. Contract storage fields of type TreeMap and DynArray are never assigned in `__init__`.

The consensus key is deliberately coarse: status, retention band, sharing, arbitration, auto-renewal, refund band, score and level. Exact day counts and quote wording are not part of it, so validators agree when they reach the same facts with different quotes.

Besides the key, each validator re-checks the leader's result on its own copy of the page: the leader's `raw` must equal `_ground(raw, my_page)` (so every quote is on my page, passes the keyword gate, and nothing else is in it), and re-running `_rate_raw(raw)` must reproduce the leader's fields, points, score, level and `why` exactly. For a challenge the same is done for the counter page and for the merge onto the stored rating. A leader therefore cannot fabricate a quote, smuggle in a different score, or store free text. Reasons for UNDETERMINED results are mapped to a fixed list, so leader text is never stored.

## URL rules

Item URLs, prefixes and counter-evidence URLs must be https, at most 300 printable ASCII characters, without whitespace or backslashes, userinfo, a port other than 443, a query string, a fragment, double slashes, dot segments or encoded dots, slashes, backslashes and NUL. The host is lowercased; the path is kept as written. A prefix must end with `/` and have a non-empty path. A URL is "under" a prefix when its canonical form starts with the prefix and is longer than it.

## Money

- Only `register_terms` and `challenge_assessment` accept value, and neither ever reverts on bad input: the value goes to the caller's pull-payment balance and the call returns `REJECTED:<reason>`.
- The contract balance always equals assurance held on open versions, plus bonds held on challenged versions, plus the sum of withdrawable balances. `get_accounting` recomputes the first two from the stored versions and compares them with the running totals and with the real balance; the tests check it after every call.
- Only `withdraw` sends GEN out. It zeroes the balance before the transfer.
- Assurance and bond are whole GEN only (assurance 0 to 100, bond exactly 2).
- Payouts: clean finalize or timeout, assurance to the registrant. Upheld challenge, bond and assurance to the challenger. Rejected challenge, bond and assurance to the registrant. Undetermined or stale challenge, bond back to the challenger.

## Trust assumptions

- The owner is trusted to approve prefixes that point at pages that cannot be edited by hostile parties, and to use one prefix per document family. The branch in a raw GitHub URL is mutable; a pinned commit path is stronger.
- A challenger can only use counter evidence under the same owner-approved prefix, so a rating can only be revised by evidence from the same publisher space.
- The model is trusted only to find quotes. It can make a field UNSTATED by being unhelpful (which costs precautionary points), but it cannot add a fact without a quote on the page, and a quote on the page that points the wrong way (for example a YES flag on a sentence that says the opposite) is caught only by validator agreement and by challenge.

## Known limits

- Registration is free, so a spammer can fill the 1000-version limit with unrated URLs under an approved prefix. The prefix owner controls which URLs exist under it, but the contract does not check that a page exists at registration.
- Pages longer than 15000 characters are cut; a clause after the cut is UNSTATED.
- The keyword gate can demote a correct but oddly worded quote to UNSTATED; that errs on the strict side by design.
- Retention and refund periods are read from one quote per field. A page that states several periods in different places is scored on the quote the model chose.
- One document type, one language: the prompt and the keyword lists are English.
