# GateHouse Administrator Guide

This guide is for administrators who manage the GateHouse platform. It covers system settings, user management, template configuration, role-based access control, and other administrative functions.

---

## Table of Contents

1. [Admin Overview](#1-admin-overview)
2. [System Settings](#2-system-settings)
3. [System Configuration](#3-system-configuration)
4. [Account Management](#4-account-management)
5. [Template Management](#5-template-management)
6. [Tools & Skills](#6-tools--skills)
7. [Role-Based Access Control (RBAC)](#7-role-based-access-control-rbac)
8. [Access Management](#8-access-management)
9. [Best Practices](#9-best-practices)
10. [Troubleshooting](#10-troubleshooting)

---

## 1. Admin Overview

### Accessing the Admin Panel

Navigate to `/admin` to access the Administration panel. You must have admin privileges to access this area.

[Screenshot: Admin panel overview]

### Admin Tab Navigation

The Admin panel uses a tabbed interface with the following sections:

| Tab | Icon | Purpose |
|-----|------|---------|
| **Settings** | Gear | General system settings and LLM provider configuration |
| **System Config** | Sliders | Low-level system configuration values |
| **Accounts** | Users | User and agent account management |
| **Agent Templates** | Bot | AI agent template configuration |
| **Project Templates** | File | Project template management |
| **Tools** | Wrench | Tool registry management |
| **Skills** | Sparkles | Skills management |
| **Roles** | Shield | Role and permission management |
| **Access** | Key | Access relationship management |

To switch tabs, click the desired tab in the navigation panel. The URL updates to reflect the active tab (e.g., `/admin?tab=accounts`).

---

## 2. System Settings

The **Settings** tab (`/admin?tab=settings`) provides high-level configuration for the platform.

[Screenshot: Settings panel with category sidebar]

### Settings Categories

Settings are organized into categories accessible via the left sidebar:

#### General
Basic system settings including:
- Application name
- Default timezone
- Session timeout

#### AI / LLM
Configure AI providers for the platform:

[Screenshot: AI/LLM provider settings]

**Supported Providers:**
- **OpenRouter**: Multi-model API gateway
- **OpenAI**: GPT models
- **Anthropic**: Claude models
- **Bedrock**: AWS-hosted models

**For each provider you can configure:**

| Setting | Description |
|---------|-------------|
| **Enable/Disable** | Toggle to activate the provider |
| **API Key** | Authentication credential (stored securely) |
| **Default Model** | Pre-selected model for new requests |
| **Base URL** | Custom API endpoint (optional) |
| **Set as Default** | Make this the default provider for new projects |

**Bedrock-specific settings:**
- AWS Region
- AWS Profile name

#### Storage
Configure file storage settings:
- Storage backend selection
- Connection parameters
- Retention policies

#### Security
Security-related settings:
- Password requirements
- Session policies
- Audit logging options

### Saving Settings

1. Make changes to any settings
2. Click **Save Changes** in the top-right
3. A success toast confirms the save

Click **Refresh** to reload settings from the server.

---

## 3. System Configuration

The **System Config** tab (`/admin?tab=system-config`) provides access to low-level configuration values.

[Screenshot: System configuration panel]

### Configuration Categories

Configurations are organized into these categories:

| Category | Description |
|----------|-------------|
| **Temporal** | Workflow timeouts and retry policies |
| **LLM** | Model defaults and cost estimation |
| **Sandbox** | Execution environment limits |
| **API** | Pagination and request limits |
| **File** | File processing settings |

### Configuration Interface

1. **Category Sidebar**: Click a category to view its configurations
2. **Search**: Filter configurations by name or description
3. **Refresh Cache**: Reload configuration from the database

### Configuration Value Types

| Type | Input | Example |
|------|-------|---------|
| **int** | Number input | `100` |
| **float** | Decimal input | `0.75` |
| **duration** | Milliseconds | `60000` (shown as "1.0 min") |
| **bool** | Toggle switch | Enabled/Disabled |
| **string** | Text input | `default-value` |
| **json** | Multi-line JSON | `{"key": "value"}` |

### Editing Configuration Values

1. Click on a configuration value (right side)
2. Enter the new value
3. Click **Save** to apply or **Cancel** to discard

**Validation:**
- Min/Max constraints are enforced for numeric values
- Invalid values show error messages
- Sensitive values cannot be edited through this interface

### Refresh Cache

After making configuration changes, click **Refresh Cache** to ensure all services pick up the new values.

### LLM Models Summary

The sidebar shows a summary of available LLM models by provider, indicating how many models are active for each.

---

## 4. Account Management

The **Accounts** tab (`/admin?tab=accounts`) manages user and agent accounts.

[Screenshot: Accounts panel with filters and data table]

### Account Types

GateHouse supports two account types:

| Type | Description |
|------|-------------|
| **User** | Human users who log in with email/password |
| **Agent** | AI agents that operate within the system |

### Filtering Accounts

Use the filter buttons to view:
- **All**: All accounts
- **Users**: Human user accounts only
- **Agents**: AI agent accounts only

Use the search box to find accounts by name or email.

### Account Statuses

| Status | Badge Color | Description |
|--------|-------------|-------------|
| **Active** | Green | Account can log in and operate |
| **Inactive** | Gray | Account is disabled but not deleted |
| **Suspended** | Red | Account is temporarily blocked |
| **Pending** | Yellow | Account awaiting activation |

### Creating a User Account

1. Click **Create User** button
2. Fill in the form:

[Screenshot: Create User dialog]

| Field | Required | Description |
|-------|----------|-------------|
| **Email** | Yes | Must be unique, used for login |
| **Display Name** | Yes | Shown throughout the application |
| **Password** | Yes | Must meet security requirements |

3. Click **Create User**

### Creating an Agent Account

1. Click **Create Agent** button
2. Fill in the form:

[Screenshot: Create Agent dialog]

| Field | Required | Description |
|-------|----------|-------------|
| **Agent Name** | Yes | Display name for the agent |
| **Description** | No | What does this agent do? |
| **Agent Template** | Yes | Select from configured templates |

3. Click **Create Agent**

### Editing an Account

1. Click on an account row or the **Edit** action in the row menu
2. You're navigated to `/admin/accounts/:id`
3. Make changes and save

### Deleting an Account

1. Click the menu (three dots) on an account row
2. Select **Delete**
3. Confirm the deletion in the dialog

**Warning**: Deleting an account removes all associated data and cannot be undone.

---

## 5. Template Management

### Agent Templates

Navigate to **Agent Templates** tab (`/admin?tab=agent-templates`) to manage AI agent configurations.

[Screenshot: Agent Templates panel]

**Agent Template Fields:**

| Field | Description |
|-------|-------------|
| **Name** | Template identifier |
| **Model** | Default LLM model to use |
| **Provider** | LLM provider (OpenRouter, Anthropic, etc.) |
| **System Prompt** | Instructions that define agent behavior |
| **Max Tokens** | Maximum response length |
| **Temperature** | Creativity setting (0.0 - 1.0) |
| **Tools** | Tools available to this agent |

**Creating an Agent Template:**

1. Click **Create Agent Template**
2. Configure the template settings
3. Optionally assign tools from the tool registry
4. Click **Create**

**Managing Agent Templates:**
- Click a template to view/edit details at `/admin/agent-templates/:id`
- Use the row menu for quick actions (Edit, Delete)

### Project Templates

Navigate to **Project Templates** tab (`/admin?tab=templates`) to manage project templates.

[Screenshot: Project Templates panel]

**Project Template Fields:**

| Field | Description |
|-------|-------------|
| **Name** | Template name shown to users |
| **Description** | Explains what the template is for |
| **Category** | Grouping for the template library |
| **Is Public** | Whether users can see this in the library |
| **Usage Count** | How many times this template was used |

**Managing Template Stages:**

Each project template can have multiple stages that define the workflow:

1. Click a template to open details at `/admin/templates/:id`
2. View the **Stages** section
3. Add, edit, or reorder stages

**Stage Properties:**
- Sequence number
- Stage name
- Description
- Stage type
- Approval required (yes/no)

**Template Visibility:**
- Set **Is Public** to `true` to show in the Template Library
- Private templates are only visible to admins

---

## 6. Tools & Skills

### Tools Panel

Navigate to **Tools** tab (`/admin?tab=tools`) to manage the tool registry.

[Screenshot: Tools panel]

Tools are capabilities that can be assigned to agents, such as:
- File operations
- Code execution
- Web requests
- Database queries

**Tool Properties:**
- Name
- Description
- Input schema
- Permissions required

Click a tool to view details at `/admin/tools/:id`.

### Skills Panel

Navigate to **Skills** tab (`/admin?tab=skills`) to manage skills.

[Screenshot: Skills panel]

Skills are higher-level capabilities that combine multiple tools or define specific workflows.

Click a skill to view details at `/admin/skills/:id`.

---

## 7. Role-Based Access Control (RBAC)

The **Roles** tab (`/admin?tab=roles`) manages roles and permissions.

[Screenshot: Roles panel with data table]

### Understanding Roles and Permissions

**Roles** are named collections of permissions that can be assigned to users, agents, or services.

**Permissions** are granular rights to perform specific actions on resources.

### Role Properties

| Property | Description |
|----------|-------------|
| **Role Key** | Unique identifier (e.g., `admin`, `editor`) |
| **Name** | Display name (e.g., "Administrator") |
| **Description** | What this role allows |
| **Scope** | Global or resource-specific |
| **Priority** | Higher priority roles take precedence |
| **Is System** | System roles cannot be deleted |

### Role Types

| Type | Badge | Description |
|------|-------|-------------|
| **System** | Blue "System" badge | Built-in roles, cannot be deleted |
| **Custom** | Gray "Custom" badge | User-created roles |

### Assignability

Roles can be configured to be assignable to:
- **Users** (human accounts)
- **Agents** (AI accounts)
- **Services** (system services)

Icons in the table indicate which types each role can be assigned to.

### Creating a Custom Role

1. Click **Create Role** button
2. Fill in the form:

[Screenshot: Create Role dialog]

| Field | Required | Description |
|-------|----------|-------------|
| **Role Key** | Yes | Unique identifier, lowercase with underscores |
| **Display Name** | Yes | Human-readable name |
| **Description** | No | Explanation of the role's purpose |
| **Priority** | No | Number (higher = more authoritative) |
| **Assignable To** | - | Toggle which account types can receive this role |

3. Click **Create Role**

### Managing Role Permissions

1. Click on a role row to open the details dialog
2. View assigned permissions in the **Permissions** section
3. Click **Add Permission** to add new permissions
4. Click the **X** on a permission to remove it

[Screenshot: Role details dialog with permissions]

**Adding Permissions:**

1. Click **Add Permission**
2. Search or browse available permissions
3. Permissions are grouped by resource type
4. Click a permission to add it to the role
5. Click **Done** when finished

**Permission Properties:**
- **Display Name**: Human-readable permission name
- **Permission Key**: Technical identifier
- **Resource Type**: What type of resource it applies to
- **Is Dangerous**: Red badge indicates elevated risk

### Deleting a Role

1. Click the menu (three dots) on a role row
2. Select **Delete** (not available for system roles)
3. Confirm the deletion

**Note**: Deleting a role removes it from all accounts that have it assigned.

---

## 8. Access Management

The **Access** tab (`/admin?tab=access`) provides a view of all access relationships in the system.

[Screenshot: Access panel with stats and table]

### Access Overview Stats

Four cards show aggregate statistics:
- **Total Relationships**: Count of all access grants
- **Unique Users**: Count of distinct users with access
- **Workspaces**: Access grants to workspaces
- **Projects**: Access grants to projects

### Resource Types

| Type | Icon | Description |
|------|------|-------------|
| **Workspace** | Folder Open | Container for projects |
| **Project** | Folder Kanban | Individual project |
| **Stage** | Layers | Workflow stage |
| **Artifact** | File | Generated output |

### Access Roles

| Role | Badge Color | Permissions |
|------|-------------|-------------|
| **Owner** | Coral | Full control including deletion |
| **Admin** | Violet | Manage team, configure settings |
| **Editor** | Blue | Create and modify content |
| **Viewer** | Gray | Read-only access |

### Filtering Access Records

- **Search**: Filter by user name, email, or resource name
- **Resource Type Filter**: Show only specific resource types

### Revoking Access

**Individual Revocation:**
1. Find the access record in the table
2. Click the trash icon on the row
3. Confirm the revocation

[Screenshot: Revoke access confirmation dialog]

**Bulk Revocation:**
1. Click **Bulk Actions** dropdown
2. Select a user to revoke all their access
3. The dialog shows how many resources will be affected
4. Click **Revoke All** to confirm

[Screenshot: Bulk revoke dialog]

### Refreshing Access Data

Click the refresh button (circular arrow) to reload access relationships from the server.

---

## 9. Best Practices

### Security

1. **Principle of Least Privilege**
   - Assign the minimum permissions needed
   - Use viewer roles for read-only access
   - Reserve admin roles for those who need them

2. **Regular Access Audits**
   - Review the Access tab periodically
   - Remove access for departed team members
   - Check for overly permissive role assignments

3. **Protect Admin Accounts**
   - Use strong, unique passwords
   - Limit the number of admin accounts
   - Consider separate admin accounts from daily-use accounts

4. **API Key Management**
   - Rotate API keys periodically
   - Use different keys for different environments
   - Never share keys in plain text

### Account Lifecycle

1. **Onboarding Users**
   - Create accounts with appropriate initial roles
   - Add to relevant workspaces and projects
   - Verify email addresses are correct

2. **Role Changes**
   - Update roles when responsibilities change
   - Document reasons for elevated privileges
   - Review inherited permissions when changing roles

3. **Offboarding**
   - Use the Access panel to find all user access
   - Consider bulk revocation for departing users
   - Delete or deactivate accounts promptly

### Template Organization

1. **Naming Conventions**
   - Use descriptive names
   - Include version or variant in name if applicable
   - Consistent naming across similar templates

2. **Categories**
   - Use meaningful categories
   - Keep category count manageable
   - Align with team or project structures

3. **Maintenance**
   - Retire outdated templates
   - Update templates when best practices change
   - Monitor usage counts to identify popular templates

### System Configuration

1. **Document Changes**
   - Note why configuration values were changed
   - Keep a changelog for significant updates
   - Test in non-production first when possible

2. **Monitor Performance**
   - Watch for impacts after configuration changes
   - Adjust timeouts based on actual workload
   - Balance resource limits with user needs

---

## 10. Troubleshooting

### Permission Errors

**Issue**: User reports "Access Denied" or "Forbidden"

**Steps:**
1. Check the user's roles in the Accounts panel
2. Verify role permissions include the needed action
3. Check resource-level access in the Access panel
4. Verify the user is accessing the correct resource

**Common Causes:**
- User was never granted access
- Role doesn't include needed permission
- Access was revoked
- Resource was moved to a different workspace

### Configuration Issues

**Issue**: Configuration changes not taking effect

**Steps:**
1. Verify the save was successful (check for error toasts)
2. Click **Refresh Cache** in System Config
3. Check if the value is within allowed min/max range
4. Restart relevant services if necessary

**Issue**: Cannot edit a configuration value

**Causes:**
- The configuration is marked as sensitive
- You don't have sufficient admin privileges
- The value has validation constraints

### Access Problems

**Issue**: Cannot see access relationships

**Steps:**
1. Verify you have admin access
2. Check if filters are hiding results
3. Try clearing search and resource type filters
4. Click refresh to reload data

**Issue**: Cannot revoke access

**Causes:**
- The access is owned by a system account
- You're trying to revoke owner access (transfer ownership first)
- Network connectivity issues

### Account Issues

**Issue**: Cannot create user account

**Common Causes:**
- Email already exists in the system
- Email format is invalid
- Password doesn't meet requirements
- Missing required fields

**Issue**: Agent account not working

**Steps:**
1. Verify the agent template exists and is properly configured
2. Check that the agent's status is "Active"
3. Verify the agent has necessary tool permissions
4. Check LLM provider configuration

### Template Issues

**Issue**: Template not appearing in library

**Causes:**
- Template's "Is Public" is set to false
- Template has no stages defined
- Template is missing required fields

**Issue**: Project from template is incomplete

**Steps:**
1. Check the template's stage configuration
2. Verify all stages have valid settings
3. Re-create the project if template was fixed

### Getting Additional Help

If issues persist:
1. Check application logs for error details
2. Review database state for inconsistencies
3. Contact system administrators
4. Report issues at https://github.com/maraichr/GateHouse/issues

---

*GateHouse Administrator Guide v1.0*
