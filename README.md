# DisputeResolver

A reusable GenLayer Intelligent Contract for neutral two-party arbitration. Each party submits one argument and one evidence URL. Validators fetch both evidence pages, an LLM weighs the case against the agreed rules, and the network reaches consensus on a ruling: `CLAIMANT`, `RESPONDENT` or `SPLIT`.

Other contracts (escrow, freelance payments, bounties, DAO grievances) can call `get_ruling(id)` and branch on the result.

## How it works

1. **`open_dispute`**: the claimant names a respondent address, describes the dispute and sets the rules the ruling must follow.
2. **`submit_argument`**: the claimant and the respondent each submit one argument and one public evidence URL. Nobody else can submit, and each side can submit only once.
3. **`rule`**: once both sides have submitted, anyone can trigger the ruling. Each validator fetches both evidence pages, an LLM judges the case against the rules, and consensus decides the result.
4. **`get_ruling`** / **`get_dispute`**: read the ruling and the stored reasoning.

## Consensus design

| Aspect | Choice |
|---|---|
| Non-deterministic block | Fetch two evidence pages, then one LLM judgment |
| Equivalence principle | `gl.eq_principle.prompt_comparative` |
| Output format | Fixed JSON: `{"ruling": ..., "reasoning": ...}` |
| Agreement rule | `ruling` must match exactly; `reasoning` may differ in wording but must not contradict it |
| Fallback | Invalid output, or an evidence page that fails to load, leads to `SPLIT` (the neutral outcome) |
| Resolution | One-shot: `OPEN` becomes `RULED`, and a ruled dispute cannot be ruled again |
| Prompt injection | Evidence pages are marked as untrusted data in the prompt, and only the agreed rules decide |
| Size limits | 5000 characters per evidence page, 1500 characters per argument |

## Contract interface

**Write methods**

| Method | Parameters | Notes |
|---|---|---|
| `open_dispute` | `respondent` (0x address as text), `description`, `rules` | Returns the new dispute id |
| `submit_argument` | `dispute_id`, `argument`, `evidence_url` | Claimant or respondent only, once each |
| `rule` | `dispute_id` | Requires both arguments; stores the ruling |

**View methods**

| Method | Returns |
|---|---|
| `get_ruling(dispute_id)` | `CLAIMANT`, `RESPONDENT` or `SPLIT` (empty until ruled) |
| `get_dispute(dispute_id)` | Full dispute as JSON, including the stored reasoning |
| `total_disputes()` | Number of disputes created |

## Using it from another contract

An escrow contract can read the ruling to decide where funds go:

```python
ruling = dispute_resolver.view().get_ruling(dispute_id)
if ruling == "CLAIMANT":
    ...  # release to claimant
elif ruling == "RESPONDENT":
    ...  # release to respondent
else:
    ...  # split
```

## Testing in GenLayer Studio

1. Upload `dispute_resolver.py` to Studio and deploy it.
2. Use two different Studio accounts: one as claimant, one as respondent.
3. As the claimant, call `open_dispute` with the respondent's full address, a description and the rules.
4. Call `submit_argument` as the claimant, switch accounts, then call `submit_argument` as the respondent.
5. Call `rule`, then `get_ruling` and `get_dispute`.

## Test results

Run in GenLayer Studio under Normal (Full Consensus) mode:

| Step | Result |
|---|---|
| Deploy | Accepted |
| `open_dispute` | Finalized |
| `submit_argument` (claimant, evidence: genlayer.com) | Finalized |
| `submit_argument` (respondent, evidence: example.com) | Finalized |
| `rule(0)` | Finalized, consensus reached |
| `get_ruling(0)` | `"RESPONDENT"` |

Stored reasoning for the test dispute: neither evidence page showed finished work, and the rules required visible finished work for a delivery to count. The ruling was reached under full consensus.

Screenshots of the run are in the `screenshots/` folder.

## Edge cases handled

- Opening a dispute against your own address is rejected
- A malformed respondent address is rejected
- Only the two parties can submit, and each only once
- `rule` cannot run until both arguments exist
- A ruled dispute cannot be ruled again
- An evidence page that fails to load does not break the ruling

## Limitations and ideas

- Evidence must be public web pages; private or login-gated pages cannot be read
- The ruling is a three-way choice; weighted splits are a possible extension
- An appeal round with a larger validator set would be a natural addition
- Fund-holding escrow can be built on top by reading `get_ruling`

## License

MIT
