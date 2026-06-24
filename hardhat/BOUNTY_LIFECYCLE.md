# AIJudge: Commit-Reveal Bounty System

This contract extends the original `AIJudge.sol` bounty system to fix a
critical flaw: submissions used to be stored on-chain in plaintext the
moment they were submitted. That meant any participant could read another
participant's answer before the deadline and submit an improved copy of it.

This version adds a **commit-reveal** flow so answers stay hidden until
after the submission window closes.

## Lifecycle

A bounty now moves through four phases:

### 1. Create
The bounty owner calls `createBounty(title, rubric, submissionWindow, revealWindow)`
and attaches the reward as `msg.value`. This sets two deadlines:
- `submissionDeadline` = now + `submissionWindow`
- `revealDeadline` = `submissionDeadline` + `revealWindow`

### 2. Commit (Submission phase)
Before `submissionDeadline`, participants call:

```
submitCommitment(bountyId, commitment)
```

`commitment` is a hash computed off-chain as:

```
commitment = keccak256(abi.encodePacked(answer, salt, msg.sender, bountyId))
```

Only the hash is stored on-chain. The plaintext answer and salt stay on the
participant's own device until the reveal phase. No one, not even the
bounty owner, can read another participant's answer during this phase.

### 3. Reveal (Reveal phase)
After `submissionDeadline` passes (and before `revealDeadline`), each
participant calls:

```
revealAnswer(bountyId, submissionIndex, answer, salt)
```

The contract recomputes the hash from the supplied `answer` and `salt` and
checks it matches the commitment stored in step 2. If it matches, the
plaintext answer is now stored on-chain and marked `revealed = true`. If a
participant never reveals, their submission stays hidden forever and is
excluded from judging.

### 4. Judge
After `revealDeadline` passes, the bounty owner assembles `llmInput`
off-chain using **only revealed submissions**, then calls:

```
judgeAll(bountyId, llmInput)
```

This calls Ritual's `LLM_INFERENCE_PRECOMPILE` exactly as in the original
contract, in a single batched call rather than one call per submission.

### 5. Finalize
Once judged, the owner calls:

```
finalizeWinner(bountyId, winnerIndex)
```

The contract requires that the winning submission was actually revealed
(`revealed == true`) before releasing the reward, so a hidden/unrevealed
submission can never win.

## Why this fixes the original flaw

| Before | After |
|---|---|
| `submitAnswer` stored plaintext immediately | `submitCommitment` stores only a hash |
| Any participant could read others' answers before the deadline | Answers stay hidden until the submitter reveals them |
| No way to penalize a no-show submission | Unrevealed commitments are simply excluded from judging and can never win |

## Required functions implemented

- `submitCommitment(uint256 bountyId, bytes32 commitment)`
- `revealAnswer(uint256 bountyId, uint256 submissionIndex, string answer, bytes32 salt)`
- `judgeAll(uint256 bountyId, bytes calldata llmInput)`
- `finalizeWinner(uint256 bountyId, uint256 winnerIndex)`

See `architecture.md` for the on-chain vs off-chain breakdown, and
`test-plan.md` for reveal-phase test cases.
