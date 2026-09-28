# AgentBank Docs

Mintlify documentation for AgentBank.

## Local preview

```bash
npx mintlify dev
```

The site has three audience-specific tabs:

- **AgentBank:** connection guides and user actions for payments, approvals,
  and account security.
- **MCP reference:** tool contracts, configuration, agent instructions, and
  technical troubleshooting.
- **Partner API:** merchant integration guides and endpoint references.

`docs.json` owns current navigation and redirects. Keep user guides focused on
what a person asks, reviews, and completes. Put execution details and schemas
in the corresponding technical reference, and link to them from guides.

## Environments

- Staging: `https://staging.agentbank.world`
- Production: `https://app.useagentbank.com`
- Partner API Sandbox: `https://staging-protocol.agentbank.world`
- Partner API Production: `https://protocol.useagentbank.com`
- Skill: `https://useagentbank.com/SKILL.md`
- Remote MCP: `https://mcp.useagentbank.com`
- Marketing site: `https://useagentbank.com`

## Agent discovery

The root `skill.md` overrides Mintlify's generated agent skill and must stay a
pointer to `https://useagentbank.com/SKILL.md` with no procedures or
tool names; the `/.well-known/mcp.json` on this host is Mintlify's
documentation-search server, not the AgentBank MCP.

## Content sources

Technical reference pages must remain aligned with the official AgentBank MCP
specification and AgentBank skill. Public guides use AgentBank terminology and
document `AGENTBANK_MCP_*` configuration for new installations.
