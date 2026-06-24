# Test Plan: Commit-Reveal Flow

This test plan covers the reveal-phase logic added to `AIJudge.sol`. It is
organized by phase, with both expected-success and expected-failure cases
for each.

## 1. Commit phase (`submitCommitment`)

| # | Scenario | Expected result |
|---|---|---|
| 1.1 | Participant submits a valid commitment before `submissionDeadline` | Succeeds, `CommitmentSubmitted` event emitted, submission stored with `revealed = false` |
| 1.2 | Participant submits after `submissionDeadline` has passed | Reverts with "submission phase over" |
| 1.3 | 11th participant tries to submit when `MAX_SUBMISSIONS` (10) already reached | Reverts with "too many submissions" |
| 1.4 | Submission attempted on a bounty that is already `judged` or `finalized` | Reverts with "already judged" / "already finalized" |
| 1.5 | Two different participants submit two different commitments to the same bounty | Both succeed, both stored at separate indices |

## 2. Reveal phase (`revealAnswer`)

| # | Scenario | Expected result |
|---|---|---|
| 2.1 | Participant reveals the correct `answer` + `salt` matching their commitment, during the reveal window | Succeeds, `answer` is stored on-chain, `revealed = true`, `AnswerRevealed` emitted |
| 2.2 | Participant reveals **before** `submissionDeadline` (reveal attempted too early) | Reverts with "submission phase not over" |
| 2.3 | Participant reveals **after** `revealDeadline` has passed | Reverts with "reveal phase over" |
| 2.4 | Participant reveals with the wrong `answer` (mismatched hash) | Reverts with "answer/salt does not match commitment" |
| 2.5 | Participant reveals with the wrong `salt` (right answer, wrong salt) | Reverts with "answer/salt does not match commitment" |
| 2.6 | Participant tries to reveal a `submissionIndex` belonging to someone else | Reverts with "not your submission" |
| 2.7 | Participant tries to reveal the same submission twice | Reverts with "already revealed" |
| 2.8 | Participant never calls `revealAnswer` at all before `revealDeadline` | Submission stays `revealed = false` permanently; excluded from judging |
| 2.9 | Revealed `answer` exceeds `MAX_ANSWER_LENGTH` | Reverts with "answer too long" |

## 3. Judging (`judgeAll`)

| # | Scenario | Expected result |
|---|---|---|
| 3.1 | Owner calls `judgeAll` after `revealDeadline`, with `llmInput` built only from revealed submissions | Succeeds, calls `LLM_INFERENCE_PRECOMPILE`, stores `aiReview`, emits `AllAnswersJudged` |
| 3.2 | Owner calls `judgeAll` before `revealDeadline` has passed | Reverts with "reveal phase not over" |
| 3.3 | Non-owner calls `judgeAll` | Reverts with "not bounty owner" |
| 3.4 | Owner calls `judgeAll` a second time on an already-judged bounty | Reverts with "already judged" |
| 3.5 | Owner calls `judgeAll` on a bounty with zero commitments at all | Reverts with "no submissions" |
| 3.6 | Owner calls `judgeAll` where all participants committed but none revealed | Reverts with "no submissions" once submission list is filtered, or precompile receives empty input (depending on off-chain assembly — should be tested explicitly) |

## 4. Finalizing (`finalizeWinner`)

| # | Scenario | Expected result |
|---|---|---|
| 4.1 | Owner finalizes with a `winnerIndex` pointing to a revealed submission | Succeeds, reward transferred to winner, `WinnerFinalized` emitted |
| 4.2 | Owner finalizes with a `winnerIndex` pointing to a submission that was **never revealed** | Reverts with "winner never revealed" |
| 4.3 | Owner finalizes before calling `judgeAll` | Reverts with "not judged yet" |
| 4.4 | Owner finalizes the same bounty twice | Reverts with "already finalized" |
| 4.5 | Non-owner calls `finalizeWinner` | Reverts with "not bounty owner" |

## 5. End-to-end happy path

1. Owner creates bounty with 1-hour submission window, 1-hour reveal window, 1 ETH reward.
2. Three participants each call `submitCommitment` with distinct commitments.
3. Time advances past `submissionDeadline`.
4. All three participants call `revealAnswer` with correct answer/salt pairs.
5. Time advances past `revealDeadline`.
6. Owner assembles `llmInput` from the three revealed answers and calls `judgeAll`.
7. Owner calls `finalizeWinner` with the index the AI judge selected.
8. Winner's balance increases by the reward amount; `bounty.finalized == true`.

## 6. Partial-reveal path (the case this feature exists for)

1. Owner creates bounty, three participants commit.
2. Only two of the three reveal before `revealDeadline`.
3. Owner calls `judgeAll` using only the two revealed answers.
4. The third (unrevealed) submission must not appear anywhere in `llmInput`
   and must not be eligible as a `winnerIndex` in `finalizeWinner` — confirm
   this explicitly reverts with "winner never revealed" if attempted.
   
