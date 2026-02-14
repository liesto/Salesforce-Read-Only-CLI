# Claude Code Read-Only Integration User - Setup Guide

A quick-start guide for setting up a read-only Salesforce Integration User for CLI-based debugging with Claude.

## What This Provides

A dedicated Integration User with read-only access that can:
- Query all object data (records)
- Retrieve all metadata (Flows, Apex, Layouts, etc.)
- Authenticate via SF CLI using JWT
- **Cannot** create, edit, or delete any data

## Overview

| Component | Value |
|-----------|-------|
| User Name | Claude Code |
| License | Salesforce Integration |
| Profile | Salesforce API Only System Integrations |
| Permission Set License | Salesforce API Integration |
| Permission Set | CLI_Read_Only_Integration |

---

## Quick Setup for Sandbox Environments

Use this when you have full CLI access to the org.

### Prerequisites

- SF CLI installed and authenticated to the target org with admin access
- Target org alias (e.g., `my-sandbox`)

### Step 1: Deploy the Permission Set

From this directory, run:

```bash
sf project deploy start \
  --metadata PermissionSet:CLI_Read_Only_Integration \
  --target-org <TARGET_ORG_ALIAS>
```

Verify deployment:

```bash
sf data query \
  --query "SELECT Id, Name, Label FROM PermissionSet WHERE Name = 'CLI_Read_Only_Integration'" \
  --target-org <TARGET_ORG_ALIAS>
```

### Step 2: Create the Integration User (Salesforce UI)

1. Navigate to **Setup > Users > New User**
2. Configure:

| Field | Value |
|-------|-------|
| First Name | `Claude` |
| Last Name | `Code` |
| Email | Your admin email |
| Username | `ccode@<your-org-domain>` (must be globally unique) |
| User License | `Salesforce Integration` |
| Profile | `Salesforce API Only System Integrations` |

3. Save the user

### Step 3: Assign Permission Set License (Salesforce UI)

1. Go to the Claude Code user record
2. Click **Permission Set License Assignments**
3. Click **Edit Assignments**
4. Enable: `Salesforce API Integration`
5. Save

### Step 4: Assign the Permission Set (Salesforce UI)

1. On the Claude Code user record, click **Permission Set Assignments**
2. Click **Edit Assignments**
3. Add: `CLI Read-Only Integration`
4. Save

### Step 5: Verify Setup

```bash
# Check user exists
sf data query \
  --query "SELECT Id, Name, Username, Profile.Name, IsActive FROM User WHERE Name = 'Claude Code'" \
  --target-org <TARGET_ORG_ALIAS>

# Check permission set assignment
sf data query \
  --query "SELECT Assignee.Name, PermissionSet.Name FROM PermissionSetAssignment WHERE Assignee.Name = 'Claude Code'" \
  --target-org <TARGET_ORG_ALIAS>

# Check permission set license assignment
sf data query \
  --query "SELECT Assignee.Name, PermissionSetLicense.DeveloperName FROM PermissionSetLicenseAssign WHERE Assignee.Name = 'Claude Code'" \
  --target-org <TARGET_ORG_ALIAS>
```

---

## Production Deployment

Use this when you don't have direct CLI access and need to perform setup manually.

### What Can Be Deployed via CLI

**Permission Set only** — If you have CLI access to prod, deploy it:

```bash
sf project deploy start \
  --metadata PermissionSet:CLI_Read_Only_Integration \
  --target-org <PROD_ORG_ALIAS>
```

### What Must Be Done in the UI

| Step | Action | Why Manual |
|------|--------|-----------|
| 1 | Create Permission Set | Can use CLI or create manually in Setup |
| 2 | Create User | User creation with Integration license requires UI |
| 3 | Assign Permission Set License | Per-user assignment, not deployable metadata |
| 4 | Assign Permission Set | Per-user assignment |
| 5 | Create External Client App | Must be done in UI |
| 6 | Configure App Policies | Part of External Client App setup |

### Manual Permission Set Creation (If No CLI Access)

If you cannot deploy via CLI, create the permission set manually:

1. **Setup > Permission Sets > New**
2. Label: `CLI Read-Only Integration`
3. API Name: `CLI_Read_Only_Integration`
4. License: (leave blank)
5. Save, then add these **System Permissions**:
   - API Enabled
   - View All Data
   - View Setup and Configuration
   - Modify Metadata Through Metadata API Functions
   - View All Users
   - View Roles and Role Hierarchy
   - View Event Log Files
   - View Public Dashboards
   - View Public Reports

### Production Checklist

- [ ] Permission Set created (CLI deploy or manual)
- [ ] User created with `Salesforce Integration` license
- [ ] Permission Set License assigned: `Salesforce API Integration`
- [ ] Permission Set assigned: `CLI Read-Only Integration`
- [ ] External Client App created
- [ ] JWT Bearer Flow enabled with certificate
- [ ] App Policies configured with profile
- [ ] Consumer Key saved
- [ ] CLI authentication tested

---

## Project Files

```
SF-CLI-ReadOnly/
├── SETUP-GUIDE.md                          # This file
├── SF-ReadOnly-Integration-User-Setup.md   # Detailed reference documentation
├── sfdx-project.json                       # SFDX project config
├── certs/
│   ├── server.key                          # Private key (KEEP SECURE)
│   └── server.crt                          # Certificate (upload to External Client App)
└── force-app/
    └── main/
        └── default/
            └── permissionsets/
                └── CLI_Read_Only_Integration.permissionset-meta.xml
```

> **Security:** The `certs/server.key` file should be kept secure and not committed to version control. Add `certs/` to `.gitignore`.

> **Tip:** You can reuse the same `server.key`/`server.crt` certificate pair across multiple orgs. Just upload the same `.crt` to each External Client App. The Consumer Key will be different per org.

---

## Permission Set Details

### Permissions Included

| Permission | Purpose |
|------------|---------|
| ApiEnabled | Required for any API access |
| ViewAllData | Read all records across all objects |
| ViewSetup | See Setup menu and configuration |
| ModifyMetadata | Retrieve metadata via Metadata API |
| ViewAllUsers | See user records |
| ViewRoles | See role hierarchy |
| ViewEventLogFiles | Required dependency for ViewAllData |
| ViewPublicDashboards | Required dependency for ViewAllData |
| ViewPublicReports | Required dependency for ViewAllData |

### Permission Set XML

**File:** `force-app/main/default/permissionsets/CLI_Read_Only_Integration.permissionset-meta.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<PermissionSet xmlns="http://soap.sforce.com/2006/04/metadata">
    <label>CLI Read-Only Integration</label>
    <description>Read-only access for SF CLI debugging. No create/edit/delete permissions.</description>
    <hasActivationRequired>false</hasActivationRequired>

    <userPermissions>
        <enabled>true</enabled>
        <name>ApiEnabled</name>
    </userPermissions>
    <userPermissions>
        <enabled>true</enabled>
        <name>ViewEventLogFiles</name>
    </userPermissions>
    <userPermissions>
        <enabled>true</enabled>
        <name>ViewPublicDashboards</name>
    </userPermissions>
    <userPermissions>
        <enabled>true</enabled>
        <name>ViewPublicReports</name>
    </userPermissions>
    <userPermissions>
        <enabled>true</enabled>
        <name>ViewAllData</name>
    </userPermissions>
    <userPermissions>
        <enabled>true</enabled>
        <name>ViewSetup</name>
    </userPermissions>
    <userPermissions>
        <enabled>true</enabled>
        <name>ModifyMetadata</name>
    </userPermissions>
    <userPermissions>
        <enabled>true</enabled>
        <name>ViewAllUsers</name>
    </userPermissions>
    <userPermissions>
        <enabled>true</enabled>
        <name>ViewRoles</name>
    </userPermissions>
</PermissionSet>
```

---

## External Client App Setup (Required for CLI Authentication)

Integration Users cannot log in via browser (`sf org login web` won't work). You must use JWT authentication, which requires an External Client App.

> **Note:** Salesforce is transitioning from Connected Apps to External Client Apps. Some orgs may not allow creating new Connected Apps via the UI. External Client Apps provide the same OAuth/JWT functionality.

### Step 1: Generate JWT Certificate

```bash
# Create certs directory
mkdir -p certs

# Generate private key
openssl genrsa -out certs/server.key 2048

# Generate certificate (valid 1 year)
openssl req -new -x509 -sha256 -key certs/server.key -out certs/server.crt -days 365 \
  -subj "/CN=ClaudeCode/O=YourOrg/C=US"
```

### Step 2: Create External Client App (Salesforce UI)

1. **Setup > External Client Apps > External Client App Manager > New External Client App**
2. Basic Info:
   - External Client App Name: `SF CLI Claude Code`
   - API Name: `SF_CLI_Claude_Code`
   - Contact Email: your email
   - Distribution State: `Local`
3. Click **Create**

### Step 3: Configure OAuth Settings

1. On the app detail page, find the OAuth configuration section
2. Enable OAuth: ✓
3. Callback URL: `http://localhost:1717/OauthRedirect`
4. OAuth Scopes - add:
   - `Manage user data via APIs (api)`
   - `Perform requests at any time (refresh_token, offline_access)`

### Step 4: Enable JWT Bearer Flow

1. Scroll to **Flow Enablement** section
2. Check **Enable JWT Bearer Flow**
3. Upload certificate: `certs/server.crt`
4. Click **Save**

### Step 5: Configure App Policies

1. Find **App Policies** section
2. Under **Select Profiles**, add: `Salesforce API Only System Integrations`
3. Click **Save**

### Step 6: Get Consumer Key

1. On the app detail page, find and copy the **Consumer Key** (Client ID)
2. Save this for the CLI command

### Step 7: Authenticate via CLI

```bash
sf org login jwt \
  --client-id <CONSUMER_KEY> \
  --jwt-key-file /path/to/certs/server.key \
  --username ccode@<your-org-domain> \
  --alias ccode-readonly \
  --instance-url https://<your-instance>.my.salesforce.com
```

**For sandboxes**, the instance URL format is typically:
```
https://<mycompany>--<sandboxname>.sandbox.my.salesforce.com
```

**For production**, the instance URL format is typically:
```
https://<mycompany>.my.salesforce.com
```

---

## Safe CLI Commands

### Allowed (Read-Only)

```bash
# Query records
sf data query --query "SELECT Id, Name FROM Account LIMIT 10" --target-org ccode-readonly

# Retrieve metadata
sf project retrieve start --metadata ApexClass --target-org ccode-readonly
sf project retrieve start --metadata Flow --target-org ccode-readonly

# Describe objects
sf sobject describe --sobject Account --target-org ccode-readonly

# List metadata types
sf org list metadata-types --target-org ccode-readonly
```

### Never Run (Write Operations)

```bash
# DO NOT use these commands with this user
sf project deploy start ...
sf data delete record ...
sf data create record ...
sf data update record ...
```

---

## Troubleshooting

| Error | Solution |
|-------|----------|
| "License doesn't allow the specified permission" | Assign the Permission Set License before the Permission Set |
| "ViewAllData depends on permission(s)..." | Ensure ViewEventLogFiles, ViewPublicDashboards, ViewPublicReports are enabled |
| JWT auth fails | Verify certificate uploaded to External Client App; check user profile is in App Policies |
| "Insufficient access rights on cross-reference id" | Some metadata types may need additional permissions |
| "External client app is not installed in this org" | Verify correct instance URL; check Distribution State is "Local" |
| "user hasn't approved this consumer" | Add the user's profile to App Policies |

---

## Security Considerations

1. **Protect the private key** — `server.key` provides direct authentication; never commit to version control
2. **Audit usage** — Review login history for the integration user periodically
3. **Test in sandbox first** — Validate the setup before deploying to production
4. **Process controls** — The primary protection against writes is disciplined CLI usage (only run retrieve/query commands)
5. **Certificate expiration** — Certificates expire after 1 year; set a reminder to regenerate

---

## License

This setup guide and associated metadata are provided as-is for setting up read-only Salesforce CLI access.
