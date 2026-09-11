# Zero Risk terminal recovery gap

This note records an observed Zero Risk lifecycle problem for later investigation. It intentionally does not prescribe a final implementation.

## Problem

A Zero Risk turn can remain active indefinitely after ChatGPT has stopped making useful progress when the required terminal completion signal never reaches the broker.

Two distinct failure modes have been observed:

1. ChatGPT Web shows `Error in message stream` after the turn has already started and completed many Codex tool calls. No further tool or final events arrive, broker pending calls are already zero, but the HTTP/browser turn remains active and Codex continues showing `Working` indefinitely.
2. ChatGPT reaches an ordinary final answer in the browser, but the Zero Risk connector does not deliver the required `codex_turn_complete`. The task work and validation are already finished, yet Codex remains in `Working` because `waitForSafeCompletion()` never resolves.

The second case was reproduced with Codex CLI 0.154.0 on `chatgpt-web/zero-risk-high` during a long tool-heavy task. ChatGPT produced a complete final summary and stated that its tool session ended before the requested final Git operations, while the Codex terminal continued to show the turn as active.

## Current contract

The Zero Risk MCP flow requires:

- `codex_turn_start` to bind the request;
- normal Codex tool activity;
- `codex_turn_complete` to deliver the final answer and terminate the turn.

If ChatGPT or the connector stops after `Sent`/bind but before `codex_turn_complete`, there is no general terminal transition. Browser turns intentionally do not use a short absolute deadline, so ownership can remain active indefinitely.

## Desired property

A Zero Risk turn that can no longer make progress should eventually become visibly recoverable or terminal instead of remaining indistinguishable from a healthy long-running `Working` turn forever.

A future design should be able to distinguish at least:

- healthy long reasoning with no recent tool calls;
- an active/bound connector;
- a connector or session that ended without terminal completion;
- an explicitly cancelled/failed turn;
- a normally completed turn.

## Safety and design constraints

Any eventual fix should preserve the existing Zero Risk safety model:

- do not inspect or mutate ChatGPT DOM to detect UI error text;
- do not blindly resend a prompt after Send has been activated or confirmed;
- do not treat generic inactivity as success or fabricate a final answer;
- do not revoke an accepted/bound capability solely because a local observer briefly disconnects;
- do not add a naive short hard timeout that kills legitimate long reasoning turns;
- preserve filesystem changes already made by the Codex turn;
- recovery should be scoped to the affected turn rather than requiring cancellation of unrelated parallel turns where possible.

## Investigation targets

When this work is picked up, inspect at least:

- `src/adapters/chatgpt-web/index.ts` around manual/Zero Risk `waitForSafeStart()` and `waitForSafeCompletion()`;
- `src/adapters/chatgpt-web/turn-broker.ts` terminal ownership, activity, retirement, and revocation states;
- `src/adapters/chatgpt-web/mcp-server.ts` connector/session lifecycle and `codex_turn_complete` delivery;
- launcher diagnostics and per-turn recovery UX;
- whether the ChatGPT connector exposes a trustworthy transport/session termination signal that can be used without reading page DOM.

Potential recovery mechanisms such as targeted failure/cancel, manual completion fallback, inactivity warnings, reconnect/reattach, or additional telemetry should be evaluated only after the lifecycle guarantees are established.