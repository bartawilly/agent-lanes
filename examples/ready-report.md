> Illustrative example. Not output from a real run; the SHAs and file names are made up.

READY-TO-PUSH 3f9c2ab | ancestry vs base 81d07e4: rc 0

Task: add idempotency key to booking checkout.

Attest: skills tdd, api-design loaded; ui not (no UI). Read: checkout spec s3, payments doc s2.

| gate      | rc |
|-----------|----|
| typecheck | 0  |
| lint      | 0  |
| unit      | 0  |
| integ     | 0  |
| build     | 0  |

Evidence: replay with same key returns first booking, 1 charge (checkout.idem.test.ts). Mutation: removed key check, test failed rc 1; restored.

Gaps: 1) key TTL 24h is a guess, needs ruling. 2) no load test of concurrent replays.

Docs: checkout spec s3 updated.

advisor: before edit, said lock per key not per user; done.

LESSON: assert the charge count, not only the response.
