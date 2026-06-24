# Architecture Note: Commit-Reveal Bounty Judge

## Where plaintext answers exist, and when

| Stage | Where the plaintext answer lives | What's on-chain |
|---|---|---|
| Before commit | Only on the participant's own device/wallet client | Nothing yet |
| Commit phase | Still only on the participant's device | A `bytes32` hash (`commitment`) and the submitter's address. No answer text, no salt. |
| Reveal phase | Sent in the `revealAnswer` transaction, becomes public the moment that transaction is mined | The plaintext `answer` and `salt` are now stored in contract storage and visible to anyone reading the chain |
| If never revealed | Plaintext never leaves the participant's device | Only the original hash remains on-chain forever; the contract has no way to recover the plaintext from it |
| Judging | The bounty owner reads the now-public revealed answers off-chain | `llmInput` (assembled off-chain) is passed into `judgeAll`, which the precompile processes and returns a result that is stored on-chain |

The key property: **during the window that matters (before the submission
deadline), no answer is ever in plaintext on-chain.** The commitment hash is
one-way — nobody can derive the answer from the hash without already
knowing the answer and salt. This is what stops the copy-and-improve
attack the assignment describes.

## On-chain vs off-chain responsibilities

**On-chain (`AIJudge.sol`):**
- Stores commitments (`bytes32` hashes) during the submission phase
- Verifies a revealed `(answer, salt)` pair against its stored commitment using `keccak256`
- Enforces phase ordering via `submissionDeadline` / `revealDeadline` timestamps
- Calls Ritual's `LLM_INFERENCE_PRECOMPILE` (address `0x0802`) via `_executePrecompile`, passing in `llmInput`
- Stores the judge's result (`aiReview`) and the finalized winner
- Enforces that only a `revealed == true` submission can be paid out as the winner

**Off-chain (bounty owner / frontend, not in this contract):**
- Generating the `salt` for each participant at commit time and giving it back to them to hold privately
- Computing `commitment = keccak256(abi.encodePacked(answer, salt, msg.sender, bountyId))` before calling `submitCommitment`
- After the reveal deadline, reading all `revealed == true` submissions from chain state (via `getSubmission`) and assembling them into the `llmInput` payload the precompile expects
- Deciding the exact prompt/rubric format sent to the LLM (the contract just forwards opaque bytes)

## How the LLM receives submissions for batch judging

`judgeAll` is called **once per bounty**, not once per submission. The
bounty owner is responsible for collecting every revealed answer (skipping
any that were never revealed) and packing them into a single `llmInput`
byte payload off-chain, alongside the bounty's `rubric`. That single payload
is what gets passed to `LLM_INFERENCE_PRECOMPILE`. The precompile call is
synchronous from the contract's perspective (`_executePrecompile` returns
the decoded result directly), so all answers are judged together in one
inference call rather than one round-trip per participant — this is what
keeps gas costs and call count bounded regardless of how many people
submitted.

## What this design does not solve (Required Track limitations)

- The salt itself is generated and held off-chain. If a participant loses
  their salt before the reveal window, their submission becomes
  permanently unrevealable (this is expected commit-reveal behavior, not a
  bug).
- This track does not use Ritual's TEE-backed execution for additional
  privacy — the Advanced Track would replace the "trust the participant to
  hold their salt safely" assumption with TEE-enforced encryption, so that
  even partial information leakage between commit and reveal isn't
  possible. That tradeoff is intentionally out of scope here.
  
