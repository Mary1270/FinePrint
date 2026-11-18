# FinePrint

Structured strictness scoring and version comparison for Terms of Service and Privacy Policy pages on GenLayer, with a registry other contracts can read.

Someone registers the URL of one version of a Terms or Privacy page. GenLayer validators fetch it and extract five facts, each with an exact quote from the page. The contract code checks every quote against the page, parses the numbers, and computes a strictness score from 0 to 10 and a level (LENIENT / STANDARD / STRICT) by fixed rules. No model ever writes the score or the level, and no model answer is accepted without a quote that occurs on the page. `compare(old_id, new_id)` then compares two final versions field by field in code, marks every change BETTER, WORSE or SAME, and stores the result in the registry.

"Strict" means harsh on the user: a higher score is worse for the person who accepts the terms.

## What the model does and what the code does

| Step | Done by |
|---|---|
| Check the URL is https, clean, and under an owner-approved prefix | code |
| Fetch the page | each validator |
| Locate five facts on the page and copy exact quotes | model |
| Drop every quote that is not on the page (case, whitespace and typographic punctuation are normalised), is shorter than 6 or longer than 300 characters, or does not contain a topical keyword (a NO answer for sharing or auto-renewal, and a NONE answer for refunds, must contain a negative word; a YES or NONE answer whose quote denies the fact, such as "we do not sell or share" or "refund within 7 days", is dropped unless the quote also has a contrast word like "but" or "except"); demote the field to UNSTATED | code |
| Parse retention and refund periods from the quotes and bucket them | code |
| Score the fields, add the points, assign the level | code |
| Agree on the fields that drive the rules, the score and the level | validators (exact match) |
| Re-check that the leader's quotes occur on the validator's own copy of the page and that the leader's result follows from its own quotes | each validator, in code |
| Compare two final versions, mark each change, store the result | code |
| Challenge, bonds, payouts, windows, accounting | code |

## Extracted fields

| Field | Model answer | What the code derives |
|---|---|---|
| Data retention | PERIOD (length in the quote), INDEFINITE, UNSTATED | SHORT (30 days or less), MEDIUM (up to 365 days), LONG, VAGUE (a period is claimed but no length can be read), INDEFINITE (also when a PERIOD quote has no length but says indefinitely). The longest duration in the quote is used and rounded up. |
| Third-party sharing | YES, NO, UNSTATED | as given |
| Mandatory arbitration clause | YES, NO, UNSTATED | as given |
| Automatic renewal | YES, NO, UNSTATED | as given |
| Refund deadline | WINDOW (length in the quote), NONE, UNSTATED | GENEROUS (14 days or more), BRIEF (shorter, rounded down), VAGUE, NONE |

A field without a verified quote is UNSTATED and receives precautionary points.

## Strictness rules

| Field | Points |
|---|---|
| Retention SHORT / MEDIUM / LONG | 0 / 1 / 2 |
| Retention VAGUE or UNSTATED | 2 |
| Retention INDEFINITE | 3 |
| Sharing NO / UNSTATED / YES | 0 / 1 / 2 |
| Arbitration NO / UNSTATED / YES | 0 / 1 / 2 |
| Auto-renewal NO / UNSTATED / YES | 0 / 1 / 1 |
| Refund GENEROUS / BRIEF, VAGUE or UNSTATED / NONE | 0 / 1 / 2 |

The maximum is 10. Score 0-3 is LENIENT, 4-6 is STANDARD, 7-10 is STRICT. A page needs at least two grounded fields, otherwise the assessment is UNDETERMINED. `get_rules()` returns the same table from the contract.

## Lifecycle

```
register_terms -> REGISTERED
REGISTERED --assess--> RATED (challenge window opens)
REGISTERED --assess, undetermined, 3rd time--> UNRATED (assurance refunded)
REGISTERED --stale, expire_version--> UNRATED (assurance refunded)
RATED --challenge_assessment (2 GEN bond)--> CHALLENGED
RATED --finalize after the window--> FINAL
CHALLENGED --resolve_challenge, level changes--> FINAL (challenger gets bond + assurance)
CHALLENGED --resolve_challenge, level holds or counter evidence does not cite the version--> FINAL (registrant gets bond + assurance)
CHALLENGED --resolve_challenge undetermined, or stale cancel_challenge--> RATED, bond refunded, window reopened
```

The registrant may attach an optional assurance of 0 to 100 whole GEN when registering. It is returned at the end of the version's life unless a challenge changes the level, in which case the challenger receives it together with their bond.

A challenge is only about evidence. The counter-evidence page must quote the full URL of the version as a complete token and sit under the prefix the version was registered under. That prefix is stored on the version at registration and never changes, so later prefix edits by the owner cannot block or widen challenges, or comparisons, for an existing version. The original page is not extracted again: fields stated on the counter page replace the same fields in the stored, consensus-accepted rating, the merged fields are re-scored by the same rules, and the challenge succeeds only if the level changes.

A challenge that ends without a verdict (counter page unreachable, model failure, stale timer) refunds the bond and reopens the challenge window for a full window length. It does not use up the version's chance to be challenged. At most 3 challenge filings are possible per version; the last undetermined one closes the window instead of reopening it.

## Comparing two versions

`compare(old_id, new_id)` is open to anyone and moves no money. Both versions must be FINAL and rated, must have been registered under the same stored prefix, and the old one must have been registered before the new one. For each of the five fields the code compares the stored points and writes BETTER (fewer points), WORSE (more points) or SAME, together with the old and new value, the old and new period in days, and a `changed` flag that is true when the value or period moved even though the points did not. The record also holds both scores and levels, the score delta, an overall verdict, a level verdict, and the counts of BETTER, WORSE and SAME fields. It is stored once per ordered pair; asking again returns the stored record. Use one prefix per document family, for example `.../acme/terms/` and `.../acme/privacy/`, so that only versions of the same document can be compared.

## Contract

`contracts/fineprint.py` (single file, deploy as is). Constructor: `challenge_window_seconds`, `stale_after_seconds` (each 60 to 604800). Suggested values: `600, 3600` for testing and `172800, 259200` for production.

| Method | Who | Purpose |
|---|---|---|
| `add_prefix(prefix, label)` / `remove_prefix(prefix)` | owner | manage the https URL prefixes new versions may be registered under; existing versions keep their stored prefix |
| `register_terms(url)` payable, 0-100 whole GEN | anyone | register one version of a terms page; invalid input is refunded to the caller's balance, not reverted |
| `assess(version_id)` | anyone | consensus extraction and rating |
| `challenge_assessment(version_id, counter_evidence_url)` payable, exactly 2 GEN | anyone but the registrant | dispute the rating |
| `resolve_challenge(version_id)` | anyone | consensus extraction of the counter evidence, merged with the stored rating |
| `cancel_challenge(version_id)` | anyone | refund a challenge that stayed unresolved past the stale limit |
| `finalize(version_id)` | anyone | close an unchallenged rating after the window |
| `expire_version(version_id)` | anyone | release a version stuck before its first rating |
| `compare(old_id, new_id)` | anyone | field-by-field comparison of two final versions, stored in the registry |
| `withdraw()` | anyone | pull payment of the caller's balance |

Views: `get_params`, `get_rules`, `get_stats`, `list_prefixes`, `list_versions(offset, limit)`, `get_version`, `find_version(url)`, `get_level(version_id)`, `get_level_by_url(url)`, `get_score(version_id)`, `meets_max_strictness(version_id, max_level)`, `get_changes(old_id, new_id)`, `list_comparisons(offset, limit)`, `get_balance`, `get_accounting`.

## Using the registry from another contract

Only FINAL ratings are exposed through `get_level`, `get_level_by_url`, `get_score` and `meets_max_strictness`. An empty string or `False` means "no final rating".

```python
registry = gl.get_contract_at(Address("0xFINEPRINT_ADDRESS"))
if not registry.view().meets_max_strictness("v1", "STANDARD"):
    raise Exception("terms are stricter than STANDARD or not rated")
changes = registry.view().get_changes("v1", "v2")
```

## Deployment

Studio address: added to this file after deployment. The contract is submitted on its own; there is no frontend in this repository. `evidence/` holds sample terms pages used for live testing. The sample prefixes are `https://raw.githubusercontent.com/Mary1270/FinePrint/main/evidence/acme/` and `https://raw.githubusercontent.com/Mary1270/FinePrint/main/evidence/beta/`.

| Sample page | Expected result |
|---|---|
| `acme/terms-2024.txt` | score 0, LENIENT |
| `acme/terms-2025.txt` | score 8, STRICT |
| `acme/terms-2025-amendment.txt` | counter evidence for `terms-2025.txt`; merged score 3, LENIENT, so a challenge is upheld |
| `acme/terms-2025-unrelated.txt` | does not cite `terms-2025.txt`, so a challenge using it is rejected |
| `acme/terms-2026.txt` | score 3, LENIENT |
| `acme/terms-thin.txt` | one field only, UNDETERMINED |
| `beta/terms.txt` | score 10, STRICT, different prefix, cannot be compared with `acme/` versions |

## Tests

```
python3 -m unittest discover -s tests -p "test_*.py"
```

No pip install is needed. The suite runs against a small offline stand-in for the GenLayer SDK (`tests/genlayer_stub.py`) with a scripted stand-in for the model. After every call it checks that the contract's real GEN balance equals its recorded obligations. It covers the URL rules, each parser, every cell of the scoring table, the validator checks against a tampered leader, all challenge paths including the bounded retry path, stuck-state exits, the compare rules, a mutation check that rewrites the contract source in 100 ways and requires the suite to fail every time, and a randomized sequence test that checks the accounting and state invariants after every call. The stand-in model cannot show how a real model behaves; that is covered by the live tests on Studio.

See `ARCHITECTURE.md` for the state machine, consensus design, accounting and trust assumptions.

## License

MIT
