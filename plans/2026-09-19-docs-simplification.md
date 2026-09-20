# Documentation simplification

Status: Implemented, validated, and approved by the user for publication.

## Outcome

Help people connect and use AgentBank without reading agent implementation
instructions. Keep builder contracts accessible in a separate MCP reference
tab and preserve the Partner API reference.

## Accepted scope

- Separate AgentBank, MCP reference, and Partner API navigation.
- Merge overlapping introductions, setup, money-in, money-out, tracking,
  security, and Partner API walkthroughs.
- Rewrite user guides around example requests, required information, review,
  approval, funding, and outcomes.
- Reconcile payment-link availability, local approval-policy tools, and
  local/remote tool boundaries with the latest supplied MCP guide.
- Retain technical tool contracts and useful agent guidance in MCP reference.
- Preserve old URLs with redirects and update internal links.

## Non-goals

No product changes, new payment capabilities, authentication tests on user
accounts, or changes to live documentation before review.

## Work areas

- Controller: navigation, introduction, setup, Remote MCP, autonomy, security,
  support, technical cross-links, integration review.
- Journey worker: Money in, Money out, Exchange, Payments, example prompts.
- Partner worker: Partner API guides and reference deduplication.

## Acceptance

- First-time users can choose the hosted chat, ChatGPT, Claude, or local agent.
- User guides avoid tool execution instructions and retain material limits.
- Every former navigation page remains available or has a valid redirect.
- No contradictory blanket claims about payment links or threshold tools.
- Technical MCP and Partner API contracts remain documented.
- Mintlify validation, broken-link checks, and diff checks pass.
- A reviewed local result is handed back before publication; the user has now
  authorized committing and pushing it.

## Validation and rollback

Run Mintlify validation and broken-link checking after all edits. Inspect the
navigation and redirects mechanically and review content against the supplied
guides. Compare the simplification against baseline 7cd344a for review or
rollback; do not discard unrelated work.

## Result

- AgentBank navigation: 38 pages, down from 116.
- MCP reference: 73 pages, including retained agent guidance and new local
  approval-policy and troubleshooting summaries.
- Partner API: 35 pages, with one canonical integration walkthrough.
- Eight redundant pages merged and removed; all eight have valid redirects.
- Mintlify build validation and broken-link checking passed.
- Navigation checks passed for file existence, unique pages, redirect targets,
  and preservation of every former navigation URL.
- Diff whitespace checks passed. Independent content review found no material
  information loss or blocker.
- Partner source has conflicting `preferred_rails` examples. The duplicate
  narrative example was removed in favor of the existing detailed endpoint
  example; no live API execution verification is claimed.
- The user reviewed the local result and authorized commit and push.
