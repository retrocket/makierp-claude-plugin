---
name: makierp-mcp-session
description: Handles MakiERP MCP startup, tenant orientation, and tenant access approval. Use when connecting to the MakiERP MCP server, starting tenant-scoped work, switching tenants, or diagnosing MCP tenant access state.
---

# MakiERP MCP Session

## Quick Start

1. Call `session_orientation` at the start of a MakiERP MCP conversation.
2. If no tenant is clear, call `tenants_list` and ask the user which tenant to use.
3. If `user_active_tenant` differs from `active_tenant`, ask before switching and use `tenant_access_request`.
4. If access is pending, tell the user to approve the prompt in MakiERP and poll `tenant_access_status`.
5. If access is rejected, expired, or the user is offline, stop tenant work and explain the next required user action.
6. Before tenant tool calls, state which tenant you will act on.

## Tool Path

- `session_orientation` shows signed-in user, online state, `user_active_tenant`, `active_tenant`, and granted tenants.
- `tenants_list` discovers tenants and whether this app already has live access.
- `tenant_access_request` opens the MakiERP approval prompt for a specific tenant.
- `tenant_access_status` polls a returned `request_id` until `approved`, `rejected`, or `expired`.

## Boundaries

- Tenant context comes from the MCP session, active MakiERP socket, or tenant access flow.
- Do not select tenant context through tool arguments.
- Do not override a tenant the user chose mid-conversation.
- Treat saved questions, dashboards, collections, exports, and tenant records as mutations.
- If a tenant-scoped tool fails for access, return to this skill before retrying.
