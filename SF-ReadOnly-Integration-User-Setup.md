# Salesforce Read-Only User Setup for SF CLI

A configuration guide for creating a read-only user for Salesforce CLI debugging and analysis with Claude.

## Overview

**Goal:** Create a user that can:
- Read all object data (records)
- Retrieve metadata (Flows, Page Layouts, Apex, etc.)
- Authenticate via SF CLI
- NOT create, edit, or delete any data

**Limitation:** Salesforce doesn't have a pure "read-only metadata" permission. The Metadata API permission enables both retrieve and deploy. We mitigate this through disciplined CLI usage (retrieve only, never deploy).

## Choose Your Path

| Approach | Pros | Cons | Authentication |
|----------|------|------|----------------|
| **Regular User** (Recommended) | Simple setup, easy CLI auth | Uses a full license | `sf org login web` |
| **Integration User** | Lower cost, no UI access (smaller attack surface) | Requires External Client App + JWT setup | `sf org login jwt` |

**If you have available user licenses**, use a Regular User — it's much simpler. The Integration User path is only worth it if you need to conserve licenses or want the extra security of an API-only user.

---

## Part 1: Permission Set Configuration

### Step 1: Create the Permission Set

1. Navigate to **Setup > Permission Sets > New**
2. Configure:
   - **Label:** `CLI Read-Only Integration`
   - **API Name:** `CLI_Read_Only_Integration`
   - **License:** (leave blank - don't select a license)
   - **Description:** `Read-only access for SF CLI debugging. No create/edit/delete permissions.`

### Step 2: System Permissions

Navigate to **System Permissions** in the permission set and enable:

| Permission | Purpose |
|------------|---------|
| API Enabled | Required for any API access |
| View All Data | Read access to all records across all objects |
| View Setup and Configuration | See Setup menu and configuration |
| Modify Metadata Through Metadata API Functions | Required to retrieve metadata (also enables deploy - mitigate via process) |
| View All Users | See user records |
| View Roles and Role Hierarchy | See role structure |

**Do NOT enable:**
- Modify All Data
- Customize Application (unless needed for specific metadata types)
- Manage Users
- Any "Create", "Edit", "Delete", or "Manage" permissions

### Step 3: Object Permissions (Optional Enhancement)

The "View All Data" permission covers record access, but you can explicitly configure object permissions if you want to be more granular:

For each object you need:
- **Read:** ✓ Enabled
- **Create:** ✗ Disabled
- **Edit:** ✗ Disabled
- **Delete:** ✗ Disabled
- **View All:** ✓ Enabled
- **Modify All:** ✗ Disabled

### Step 4: Field-Level Security

"View All Data" grants read access to all fields. No additional FLS configuration needed for read-only use.

---

## Part 2: User Setup

### Option A: Regular User (Recommended)

The simplest approach — use a standard Salesforce user with the read-only permission set.

#### Step 1: Create or Identify the User

Either create a new user or use an existing one:

1. Navigate to **Setup > Users > New User** (or edit existing)
2. Configure:
   - **User License:** `Salesforce` (or `Salesforce Platform`, etc.)
   - **Profile:** Any standard profile (e.g., `Standard User`, `Minimum Access - Salesforce`)

#### Step 2: Assign the Permission Set

1. On the user record, click **Permission Set Assignments**
2. Click **Edit Assignments**
3. Add: `CLI Read-Only Integration`
4. Save

#### Step 3: Authenticate via CLI

```zsh
# Opens browser for OAuth login - no Connected App needed
sf org login web \
  --alias sf-readonly \
  --instance-url https://yourorg.my.salesforce.com
```

That's it! Skip to **Part 3: Safe CLI Commands**.

---

### Option B: Integration User (Advanced)

Use this if you want to conserve licenses or prefer an API-only user with no UI access. **Requires additional Connected App setup.**

#### Step 1: Create the Integration User

1. Navigate to **Setup > Users > New User**
2. Configure:
   - **First Name:** `CLI`
   - **Last Name:** `Integration`
   - **Alias:** `cliint`
   - **Email:** your admin email (for notifications)
   - **Username:** `cli-readonly@yourorg.com` (must be unique across all Salesforce)
   - **User License:** `Salesforce Integration`
   - **Profile:** `Salesforce API Only System Integrations`

#### Step 2: Assign Permission Set License

**Critical step** - without this, permission set assignment fails.

1. Go to the user record
2. Click **Permission Set License Assignments**
3. Click **Edit Assignments**
4. Enable: `Salesforce API Integration`
5. Save

#### Step 3: Assign the Permission Set

1. On the user record, click **Permission Set Assignments**
2. Click **Edit Assignments**
3. Add: `CLI Read-Only Integration`
4. Save

#### Step 4: Generate Certificate for JWT Auth

```zsh
# Create certs directory
mkdir -p certs

# Generate private key
openssl genrsa -out certs/server.key 2048

# Generate certificate (valid 1 year)
openssl req -new -x509 -sha256 -key certs/server.key -out certs/server.crt -days 365 \
  -subj "/CN=ClaudeCode/O=YourOrg/C=US"

# Keep server.key secure - this authenticates as your integration user
```

#### Step 5: Create an External Client App

Integration Users can't log in via browser, so you need JWT authentication via an External Client App.

> **Note:** Salesforce is transitioning from Connected Apps to External Client Apps. Some orgs may not allow creating new Connected Apps via the UI.

1. Navigate to **Setup > External Client Apps > External Client App Manager**
2. Click **New External Client App**
3. Configure Basic Info:
   - **External Client App Name:** `SF CLI Claude Code`
   - **API Name:** `SF_CLI_Claude_Code`
   - **Contact Email:** your email
   - **Distribution State:** `Local`
4. Click **Create**

#### Step 6: Configure OAuth Settings

On the app detail page:

1. Enable OAuth: ✓
2. **Callback URL:** `http://localhost:1717/OauthRedirect`
3. **OAuth Scopes** - add to Selected:
   - `Manage user data via APIs (api)`
   - `Perform requests at any time (refresh_token, offline_access)`

#### Step 7: Enable JWT Bearer Flow

1. Scroll to **Flow Enablement** section
2. Check **Enable JWT Bearer Flow**
3. Upload certificate file: `certs/server.crt`
4. Click **Save**

#### Step 8: Configure App Policies

1. On the app detail page, go to the **Policies** tab
2. Click **Edit** next to App Policies
3. Change **App Authorization** to `Admin approved users are pre-authorized`
4. Add profile: `Salesforce API Only System Integrations` to the authorized profiles
5. Save

> **Why not "All users can self-authorize"?** Self-authorization requires the user to approve the app via browser login. Integration Users can't log in via browser, so they must be pre-authorized by an admin.

#### Step 9: Get Consumer Key and Authenticate

1. On the app detail page, copy the **Consumer Key** (Client ID)
2. Authenticate via CLI:

```zsh
sf org login jwt \
  --client-id YOUR_CONSUMER_KEY \
  --jwt-key-file /path/to/certs/server.key \
  --username ccode@yourorg.com \
  --alias ccode-readonly \
  --instance-url https://yourorg.my.salesforce.com
```

### Verify Connection (Either Option)

```zsh
sf org display --target-org sf-readonly
```

---

## Part 3: Safe CLI Commands for Debugging

### Allowed (Read-Only) Operations

```zsh
# Retrieve metadata
sf project retrieve start --metadata ApexClass --target-org sf-readonly
sf project retrieve start --metadata Flow --target-org sf-readonly
sf project retrieve start --metadata Layout --target-org sf-readonly

# Query records
sf data query --query "SELECT Id, Name FROM Account LIMIT 10" --target-org sf-readonly

# Describe objects
sf sobject describe --sobject Account --target-org sf-readonly

# List metadata
sf org list metadata-types --target-org sf-readonly

# Retrieve full metadata package
sf project retrieve start --manifest manifest/package.xml --target-org sf-readonly
```

### NEVER Run These Commands

```zsh
# DO NOT deploy
sf project deploy start ...

# DO NOT delete
sf data delete record ...
sf data delete bulk ...

# DO NOT create/update records
sf data create record ...
sf data update record ...

# DO NOT run destructive changes
sf project deploy start --manifest destructiveChanges.xml ...
```

---

## Part 4: Sample package.xml for Metadata Retrieval

Create `manifest/package.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Package xmlns="http://soap.sforce.com/2006/04/metadata">
    <types>
        <members>*</members>
        <name>ApexClass</name>
    </types>
    <types>
        <members>*</members>
        <name>ApexTrigger</name>
    </types>
    <types>
        <members>*</members>
        <name>Flow</name>
    </types>
    <types>
        <members>*</members>
        <name>Layout</name>
    </types>
    <types>
        <members>*</members>
        <name>CustomObject</name>
    </types>
    <types>
        <members>*</members>
        <name>CustomField</name>
    </types>
    <types>
        <members>*</members>
        <name>ValidationRule</name>
    </types>
    <types>
        <members>*</members>
        <name>WorkflowRule</name>
    </types>
    <types>
        <members>*</members>
        <name>PermissionSet</name>
    </types>
    <types>
        <members>*</members>
        <name>Profile</name>
    </types>
    <version>59.0</version>
</Package>
```

---

## Troubleshooting

### "Insufficient access rights on cross-reference id"
- The user may not have access to that metadata type
- Try adding `Customize Application` permission (increases risk slightly)
- Integration User licenses have more restrictions than regular licenses

### "This license doesn't allow the specified permission" (Integration User only)
- Ensure you assigned the **Permission Set License** before the permission set
- The `Salesforce API Integration` PSL must be assigned first

### JWT Authentication Fails (Integration User only)
- Verify the certificate is uploaded to the External Client App
- Check that the integration user's profile is added in App Policies
- Ensure the private key matches the uploaded certificate
- Verify "Enable JWT Bearer Flow" is checked
- Use the correct instance URL (check with `sf org display --target-org existing-alias`)

### "User doesn't have access to this org"
- User must be active
- For Integration Users: Check External Client App policies include the user's profile

### "External client app is not installed in this org"
- Ensure you're using the correct instance URL
- Verify the app's Distribution State is set to "Local"
- Check that App Policies include the correct profile

---

## Security Considerations

1. **Store credentials securely** - For Integration Users, `server.key` authenticates directly; protect it
2. **Audit usage** - Review login history for the read-only user periodically
3. **Principle of least privilege** - Only add permissions you actually need
4. **Sandbox first** - Test this configuration in a sandbox before production
5. **Process controls** - The main protection against writes is process discipline (only run retrieve commands)

---

## Quick Reference Card

### Authentication

| User Type | Command |
|-----------|---------|
| Regular User | `sf org login web --alias sf-readonly` |
| Integration User | `sf org login jwt --client-id X --jwt-key-file key --username user --alias sf-readonly` |

### Common Operations

| Task | Command |
|------|---------|
| Query data | `sf data query --query "SELECT ..." --target-org sf-readonly` |
| Retrieve metadata | `sf project retrieve start --metadata Type --target-org sf-readonly` |
| Describe object | `sf sobject describe --sobject ObjectName --target-org sf-readonly` |
| List metadata | `sf org list metadata-types --target-org sf-readonly` |
| Verify connection | `sf org display --target-org sf-readonly` |
