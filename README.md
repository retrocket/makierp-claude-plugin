# MakiERP for Claude

[MakiERP](https://makierp.com/) is a cloud ERP for companies in Türkiye and the TRNC: accounting, current accounts (cari), invoicing, stock and warehouses, cheques, banks, orders, pricing and e-commerce. This plugin connects Claude to your MakiERP account. You can ask about your own company data in plain language and prepare everyday entries without leaving the conversation.

The skills are written in English and tuned for Turkish-speaking businesses. They use Turkish ERP terms (cari, çek bordrosu, depo fişi, ödeme planı), and Claude answers in the language you write in.

## What you can do

- **Ask about your data:** customer and supplier balances, sales and purchase totals, stock levels and movements, collections, overdue receivables.
- **Build reports:** saved questions, charts, dashboards, Excel/PDF exports and scheduled report emails.
- **Prepare entries for approval:** bank statement lines, cheque rolls, current-account slips, credit card block clearings, payment plans, sales and purchase orders, and warehouse slips. You can start from a pasted list, a spreadsheet or a statement.
- **Investigate pricing:** why a line came out at a given price, which price cards or campaigns conflict, and where margins are thin.
- **Run your online shop:** shop settings, the order inbox, product content and images.
- **Design print templates:** invoices, statements and other printed documents.
- **Look up how MakiERP works:** fields, statuses and documented behavior.

## Requirements

- A MakiERP account with access to at least one company (tenant). The plugin does nothing without one. To get an account, contact us at [makierp.com/iletisim](https://makierp.com/iletisim).
- Claude with plugin support: claude.ai, Claude Desktop, Cowork or Claude Code.

## Install

From the Claude directory, search for **MakiERP** and add it.

In Claude Code you can also install straight from this repository:

```
/plugin install makierp --marketplace retrocket/makierp-claude-plugin
```

## How it connects

The plugin adds one remote MCP server, `https://mcp.makierp.com/`, operated by MakiERP. It has no scripts, hooks or local programs. Everything else in the plugin is skill text that teaches Claude how to use the server well.

1. **Sign in.** When Claude first calls the server, you sign in to MakiERP through OAuth. Claude never sees your password.
2. **Approve the company.** Before Claude touches any company's data, the server asks for access to that specific company. You approve the request inside MakiERP and choose how long the access lasts, or you reject it. Access expires on its own.
3. **Your permissions apply.** Claude can only see and do what your MakiERP user is allowed to see and do. Role permissions are enforced on the server for every call.

## Safety

- Read and write operations are separate tools. Every write tool is marked so Claude asks before running it.
- Entries are previewed before they are saved. Cheque rolls, current-account slips, card clearings, payment plans, accounting postings and print templates work in two steps: the first call saves nothing and returns a preview, and the server saves only when Claude repeats the call with the one-time confirmation code from that preview, after you approve it. Bank transactions, orders and warehouse slips have their own preview tools, and the skills tell Claude to show you the preview before creating anything.
- The tools record bookkeeping entries in your ERP. They do not move money, contact banks or make payments.
- Every change made through Claude is recorded in MakiERP's audit log, tagged with the AI client that made it.

## Data handling

- **What is sent:** the requests Claude makes on your behalf (search terms, filters, the entries you ask it to prepare) go only to `https://mcp.makierp.com/`. The responses contain the MakiERP data needed to answer you, and they appear in your Claude conversation.
- **Where it goes:** to no one else. The plugin and the server send nothing to other third parties. The server does not read your Claude chat history, memory or files.
- **What is stored:** your company data stays in MakiERP as it does today. MakiERP also keeps the OAuth session, the company access approvals and audit records of changes made through Claude. Retention follows the [MakiERP Privacy Policy](https://makierp.com/gizlilik-politikasi).
- **Who it is for:** business users. The plugin is not intended for people under 18.

## Support

- Contact: [makierp.com/iletisim](https://makierp.com/iletisim) or info@makierp.com
- [Privacy Policy](https://makierp.com/gizlilik-politikasi)
- [Terms of Use](https://makierp.com/kullanim-kosullari)

## License

Copyright © Retrocket Software Ltd. All rights reserved. See [LICENSE](LICENSE).
