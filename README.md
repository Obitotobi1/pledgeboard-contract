# PledgeBoard (Intelligent Contract)

Say it in public, then prove it. PledgeBoard lets anyone publish a pledge with a deadline and an evidence link. When the pledge is done, GenLayer validators read the evidence page and confirm it **only if they can quote proof from the page**. Give up, and you mark the pledge **broken** in public. Every wallet builds a track record of pledges kept and broken that any contract can read.

Promises are cheap and nobody checks them: "audit coming next month", "open source soon", "I will ship by Friday". PledgeBoard makes a promise checkable against the real web, with many independent validators agreeing, and keeps the history.

- **Contract (GenLayer Studio):** `0x09fF56b247Dd2ce742FB4188128Eff9cb542a7E3`
- **Contract source:** [`pledgeboard.py`](./pledgeboard.py)
- **Tests:** [`tests/test_pledgeboard.py`](./tests/test_pledgeboard.py)

## How it works

```
make_pledge(pledge, deadline, evidence_url) --> OPEN --claim_kept--> KEPT (with quote)
                                                  |
                                                  +--mark_broken--> BROKEN (owner admits it)
```

1. `make_pledge` publishes the pledge (one sentence, a `YYYY-MM-DD` deadline, a public https evidence page).
2. When it is done, the owner updates the evidence page and calls `claim_kept(pledge_id)`.
3. Every validator fetches the page (`gl.nondet.web.render`, first 6,000 characters), asks its model whether the page shows the pledge fulfilled, and they agree with `gl.eq_principle.prompt_comparative`: the verdict (`KEPT` or `NOT_YET`) must be identical, the quote wording may differ.
4. `KEPT` is accepted only if its quote really appears on the page. A page that only repeats the promise does not count.
5. Anyone can read the public board (`get_recent`), one pledge (`get_pledge`), a wallet's record (`get_record`) or `is_kept(pledge_id)`.

## What it protects against

| Risk | How PledgeBoard handles it |
|---|---|
| Claiming a pledge is kept with no proof | `KEPT` needs a quote that really appears in the fetched page, otherwise the result is `NOT_YET`. |
| A page that only repeats the promise | The prompt says planned or coming-soon text is not proof, and the quote rule backs it up. |
| Prompt injection from the pledge or the page | Each is fenced in its own tag with `<` and `>` escaped, and the prompt treats both as untrusted data. |
| Someone else closing or breaking your pledge | `claim_kept` and `mark_broken` are owner-only. |
| Validators fetching internal addresses | Only `https` links to public hosts. `localhost`, private and link-local IPs, `.local`/`.internal`, credentials and IPv6 literals are refused. |
| Spam and endless retries | 5 checks per pledge, 20 pledges per wallet, no duplicate open pledge, bounded text lengths and page size. |
| Rewriting history | `KEPT` and `BROKEN` are final. |

## Read API (composable)

- `get_recent(limit)` returns the newest pledges from everyone (the public board)
- `get_pledge(pledge_id)` returns one pledge as JSON (status, evidence link, quote)
- `get_pledges_by_owner(owner)` returns a wallet's pledges
- `get_record(owner)` returns `{kept, broken, open}` for a wallet
- `is_kept(pledge_id)` is `true` only for a validator-confirmed pledge

## Tests

`tests/test_pledgeboard.py` runs the contract's real logic against a small stand-in for the `genlayer` library, with the web fetch replaced by a dictionary of pages and the model by a scripted reply. No node is needed:

```
python3 tests/test_pledgeboard.py
```

It covers link and date refusal, duplicate and cap rules, accepted and invented quotes, garbage model output, unreachable pages, attempt caps, owner-only actions, final states, prompt fencing, page truncation and the read API (12 tests).

## Limits

- The contract has no clock, so the deadline is shown to everyone and given to the validators but is **not enforced on-chain**. A pledge can be claimed before or after its deadline.
- Proof is only as good as the evidence page: use a page only you can edit.
- Pages that need a login or are mostly images are not supported.
- Studio test network only. No payments or enforcement. A web app for it lives in its own project repository.
