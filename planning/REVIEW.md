# Review of `planning/PLAN.md`

## Findings

1. High: The plan depends on an agent skill that is not part of the documented project contract and does not appear to exist in this workspace/session. Section 9 instructs implementers to use a `cerebras-inference` skill for LiteLLM/OpenRouter calls, but the plan otherwise presents itself as a self-contained build spec. That makes the implementation path ambiguous and non-portable for anyone who only has the repo. Either replace this with concrete library/runtime requirements in the plan or move the agent-specific instruction out of the product spec. Reference: `planning/PLAN.md:284-295`.

2. High: The plan contradicts itself on the current repository state, which makes it unreliable as an execution source of truth. Section 13 says the backend currently contains only `app/market/` and an empty `__init__.py`, but the repo already contains `backend/README.md`, `backend/pyproject.toml`, `backend/uv.lock`, and a populated `backend/tests/` tree. That mismatch is a process risk because agents may optimize around a false starting point. Reference: `planning/PLAN.md:462-464`.

3. Medium: The data model is over-specified for a single-user SQLite app and adds avoidable implementation complexity without a matching user requirement. UUID primary keys on every table, `user_id` on every table, and fractional shares all push more validation, indexing, and rounding work into the first version, while the UX only requires a simulated single-user trading demo. This is likely to slow delivery and increase bugs in portfolio math. Reference: `planning/PLAN.md:196-240`.

4. Medium: The trade execution rules are underspecified for stale market data, especially in the Massive polling mode. The plan says trades fill at the "current price", while the real-data path may only refresh every 15 seconds on the free tier. Without an explicit freshness threshold or UI disclosure, the backend and frontend can each implement different meanings of "current", leading to surprising fills and test instability. Reference: `planning/PLAN.md:161-164`, `planning/PLAN.md:176-180`, `planning/PLAN.md:292-299`.

5. Medium: The plan creates redundant data paths for watchlist pricing and does not clearly define the source of truth. Section 8 says `GET /api/watchlist` returns current tickers with latest prices, while Section 6 already defines SSE as the live pricing channel and the frontend accumulates sparkline data from that stream. Keeping both paths in sync adds unnecessary state-merging logic and failure modes during reconnects. Reference: `planning/PLAN.md:176-180`, `planning/PLAN.md:266-268`, `planning/PLAN.md:355-356`.

6. Medium: Several operational limits are left unspecified even though they materially affect correctness and cost. The plan does not define chat-history truncation, retention/pruning for `portfolio_snapshots`, or the acceptable chart-history window. Those are not polish details; they directly shape database growth, prompt size, and what the API must return. Reference: `planning/PLAN.md:228-232`, `planning/PLAN.md:293-299`, `planning/PLAN.md:358`, `planning/PLAN.md:447-456`.

## Open Questions

- Is fractional trading actually required in the first release, or would integer shares be acceptable?
- Should "current price" mean "latest cached price regardless of age" or "latest cached price only if newer than N seconds"?
- Does the P&L chart need cross-session history, or only in-session history?
- Should the watchlist REST endpoint return prices at all, or only ticker membership?

## Summary

The strongest issue is that the document mixes product requirements with agent/tooling instructions and also misstates the current repo state. After that, the main risk is scope creep from speculative schema design and undefined runtime constraints. Tightening those areas would make the plan much easier to implement consistently.
