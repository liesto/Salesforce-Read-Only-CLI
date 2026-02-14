# Salesforce Read-Only CLI

A ready-to-use setup for creating a read-only Salesforce CLI connection using an Integration User with JWT authentication. Designed to be used with [Claude Code](https://docs.anthropic.com/en/docs/claude-code) for safe, read-only Salesforce debugging and analysis.

## What You Get

A dedicated Integration User that can:
- Query all object data (records)
- Retrieve all metadata (Flows, Apex, Layouts, etc.)
- Authenticate via SF CLI using JWT
- **Cannot** create, edit, or delete any data

## Quick Start

1. Clone this repo
2. Follow the [Setup Guide](SETUP-GUIDE.md) — 11 steps, mostly in the Salesforce UI
3. Authenticate via CLI and start querying

## What's Included

| File | Purpose |
|------|---------|
| [SETUP-GUIDE.md](SETUP-GUIDE.md) | Step-by-step setup instructions |
| [SF-ReadOnly-Integration-User-Setup.md](SF-ReadOnly-Integration-User-Setup.md) | Detailed reference documentation |
| `force-app/` | Deployable permission set metadata |
| `sfdx-project.json` | SFDX project configuration |

## Prerequisites

- [Salesforce CLI](https://developer.salesforce.com/tools/salesforcecli) installed
- Admin access to the target Salesforce org
- OpenSSL available on your machine

## Example Usage

```bash
# Query records
sf data query --query "SELECT Id, Name FROM Account LIMIT 10" --target-org sf-readonly

# Retrieve metadata
sf project retrieve start --metadata ApexClass --target-org sf-readonly

# Describe an object
sf sobject describe --sobject Account --target-org sf-readonly
```

## Security

- The `certs/server.key` file generated during setup provides direct authentication — keep it secure and never commit it
- The included `.gitignore` excludes private keys by default
- The primary protection against writes is disciplined CLI usage (only run query/retrieve commands)

## License

MIT
