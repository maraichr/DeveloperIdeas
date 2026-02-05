# GateHouse User Guide

Welcome to GateHouse, a workflow orchestration platform for AI-powered code generation and project management. This guide covers everything you need to know as a user of the platform.

---

## Table of Contents

1. [Getting Started](#1-getting-started)
2. [Managing Projects](#2-managing-projects)
3. [AI Chat](#3-ai-chat)
4. [Workflows](#4-workflows)
5. [Templates](#5-templates)
6. [Artifacts](#6-artifacts)
7. [Account & Settings](#7-account--settings)
8. [Tips & Troubleshooting](#8-tips--troubleshooting)

---

## 1. Getting Started

### Introduction to GateHouse

GateHouse is a comprehensive platform that combines AI-powered code generation with workflow orchestration. It enables you to:

- Create and manage software projects with AI assistance
- Design visual workflows that automate complex tasks
- Chat with AI agents that understand your project context
- Generate and manage code artifacts
- Collaborate with team members

### Key Concepts

| Concept | Description |
|---------|-------------|
| **Projects** | Container for all work related to a specific software initiative. Projects contain workflows, artifacts, chats, and team members. |
| **Workflows** | Visual, node-based automations that define how tasks are executed. Workflows can include AI agents, human approvals, and conditional logic. |
| **Templates** | Pre-built project and workflow configurations that help you get started quickly. |
| **Artifacts** | Generated outputs from workflows or AI interactions, such as code files, documentation, or configuration. |
| **AI Chat** | Conversational interface to interact with AI agents who understand your project context and can assist with development tasks. |
| **Workspaces** | Organizational containers that group related projects together. |

### Logging In

1. Navigate to the GateHouse login page (`/login`)
2. Enter your email address and password
3. Click **Sign In**

After successful authentication, you'll be redirected to the Dashboard.

### Navigating the Dashboard

The Dashboard (`/`) is your home screen and provides an overview of your activity.

[Screenshot: Dashboard overview showing stats cards and recent projects]

**Dashboard Elements:**

- **Welcome Message**: Personalized greeting with your display name
- **Stats Cards**: Quick metrics showing:
  - Total Projects
  - Active Projects
  - Completed Projects
- **Recent Projects**: List of your 5 most recently updated projects with:
  - Project name
  - Last updated time
  - Current status badge
- **New Project Button**: Quick access to create a new project

**Main Navigation** (sidebar):
- Dashboard (Home)
- Projects
- Templates
- Workflows
- Notifications
- Profile
- Settings
- Admin (if you have admin access)

---

## 2. Managing Projects

### Projects List

Navigate to **Projects** (`/projects`) to see all your projects.

[Screenshot: Projects list page with search and filter]

**Features:**
- **Search**: Type in the search box to filter projects by name or description
- **Status Filter**: Filter by project status:
  - All Status
  - Draft
  - Planning
  - In Progress
  - In Review
  - Completed
  - Archived
- **Project Cards**: Each project displays:
  - Project icon and name
  - Status badge (color-coded)
  - Description preview
  - Last updated time

### Creating a Project

1. Click the **New Project** button
2. The Create Project Wizard opens with options:
   - **Start from scratch**: Create a blank project
   - **Use a template**: Select from available templates

[Screenshot: Create Project Wizard]

**Project Creation Fields:**
- **Name** (required): Give your project a descriptive name
- **Description**: Explain the project's purpose
- **Workspace**: Select an existing workspace or create a new one
- **Template** (optional): Pre-populate with a template configuration

3. Click **Create** to finish

### Project Detail View

Click any project to open its detail page (`/projects/:projectId`).

[Screenshot: Project detail page with tabs]

The project detail page includes a header with:
- Project name and description
- Status badge
- Progress indicator (percentage complete)
- Back to Projects link

**Project Tabs:**

#### Overview Tab
Displays project summary including:
- Progress percentage and visual indicator
- Team members with avatars
- Pending approvals count
- Completed vs. total stages

#### Workflows Tab
Shows all workflows associated with this project:
- List of active and past workflow runs
- Status of each workflow
- Quick actions to view or re-run workflows

#### Chat Tab
Access AI conversations for this project (see [AI Chat](#3-ai-chat) section)

#### Artifacts Tab
Browse all generated artifacts (see [Artifacts](#6-artifacts) section)

#### Activity Tab
View project activity timeline showing:
- Recent changes
- Workflow executions
- Artifact generation events

#### Settings Tab
Configure project settings:
- Edit project name and description
- Manage team members
- Configure project agents
- Delete project (with confirmation)

### Sharing Projects and Managing Team Access

From the **Settings** tab of a project:

1. **View Team Members**: See current members with their roles (Owner, Admin, Editor, Viewer)
2. **Add Members**:
   - Click **Add Member**
   - Enter the user's email
   - Select a role
   - Click **Invite**
3. **Change Roles**: Use the dropdown next to each member to change their role
4. **Remove Members**: Click the remove button next to a member

**Role Permissions:**
| Role | View | Edit | Manage Team | Delete |
|------|------|------|-------------|--------|
| Viewer | Yes | No | No | No |
| Editor | Yes | Yes | No | No |
| Admin | Yes | Yes | Yes | No |
| Owner | Yes | Yes | Yes | Yes |

---

## 3. AI Chat

The AI Chat feature allows you to have conversations with AI agents who understand your project context.

### Accessing Chat

From a project detail page, click the **Chat** tab.

[Screenshot: Chat interface showing conversation list and chat panel]

### Chat Interface Layout

The chat interface has three panels:

1. **Conversation Sidebar** (left):
   - List of all conversations for this project
   - Conversation count
   - New Chat button (+)
   - Each conversation shows: title, message count, last activity

2. **Chat Panel** (center):
   - Active conversation messages
   - Message input at the bottom
   - AI responses with citations and references

3. **Context Sidebar** (right):
   - Available artifacts for reference
   - Codebase link status
   - Quick artifact selection

### Starting a New Conversation

1. Click the **+** button or **New Chat** in the conversation sidebar
2. A dialog appears to select an agent:
   - **Default Assistant**: General-purpose AI with project context and codebase tools
   - **Project Agents**: Specialized agents configured for this project (if available)

[Screenshot: New conversation dialog with agent selection]

3. Select an agent and the conversation starts immediately

### Understanding Agents

**Default Assistant:**
- Has access to your project context
- Can read and reference artifacts
- Uses codebase tools if linked
- General-purpose coding and planning assistance

**Project Agents:**
- Custom-configured by project admins
- May have specific system prompts for specialized tasks
- Can use different AI models
- Tailored capabilities for project needs

### Using Context in Chat

The AI can reference:
- **Artifacts**: Click an artifact in the right sidebar to reference it in your conversation
- **Codebase**: If a codebase is linked, the AI can search and reference code
- **Previous Messages**: Context from the current conversation is maintained

### Managing Conversations

- **Select Conversation**: Click any conversation in the left sidebar
- **View Details**: See message count and last activity time
- Conversations are auto-saved and persist across sessions

---

## 4. Workflows

Workflows are visual automations that orchestrate AI agents, human approvals, and automated tasks.

### Workflows List

Navigate to **Workflows** (`/workflows`) to see all workflows.

[Screenshot: Workflows list page]

**Workflow Card Information:**
- Workflow name and icon
- Status badge:
  - **Draft**: Work in progress, not executable
  - **Published**: Ready to run
  - **Retired**: No longer active
- Version number
- Last updated time
- Node count

**Actions Menu** (three dots):
- **Edit**: Open in workflow builder
- **View Runs**: See execution history
- **Clone**: Create a copy of this workflow
- **Delete**: Remove the workflow (with confirmation)

### Creating a Workflow

1. Click **New Workflow** on the Workflows page
2. A new draft workflow is created automatically
3. You're redirected to the Workflow Builder

### Workflow Builder

The Workflow Builder (`/workflows/:workflowId/edit`) is a visual editor for designing workflows.

[Screenshot: Workflow Builder interface]

**Interface Components:**

1. **Header Bar**:
   - Back button to Workflows list
   - Workflow name (click to edit)
   - Unsaved changes indicator
   - Undo/Redo buttons
   - Save button
   - Test Run button
   - More options menu (View Runs, Publish, Retire)

2. **Node Palette** (left sidebar):
   - Draggable node types organized by category
   - Search/filter nodes

3. **Canvas** (center):
   - Visual workflow diagram
   - Drag nodes from palette
   - Connect nodes by dragging from output to input handles
   - Click nodes to select and configure

4. **Config Panel** (right sidebar, when node selected):
   - Node-specific configuration
   - Label editing
   - Parameter settings

### Node Types

| Node Type | Purpose |
|-----------|---------|
| **Workflow Trigger** | Entry point that starts the workflow |
| **LLM Agent** | AI-powered task execution |
| **Human Approval** | Pause workflow for human review |
| **Conditional** | Branch based on conditions |
| **Parallel Split** | Execute multiple branches simultaneously |
| **Loop Start/End** | Iterate over collections |
| **Code Execute** | Run custom code |
| **HTTP Request** | Make API calls |
| **File Operations** | Read/write files |

### Building a Workflow

1. **Add Nodes**: Drag nodes from the palette onto the canvas
2. **Connect Nodes**:
   - Click and drag from a node's output handle (right side)
   - Drop on another node's input handle (left side)
3. **Configure Nodes**:
   - Click a node to select it
   - Configure in the right panel
4. **Save**: Click **Save** to persist changes

### Running Workflows

1. Click **Test Run** in the workflow builder
2. A dialog appears to enter input variables (if any are defined)
3. Click **Run** to start execution
4. Watch the canvas for real-time execution status:
   - Nodes light up as they execute
   - Progress indicators show current activity
   - Errors are highlighted in red

### Workflow Execution Controls

During execution, additional controls appear:
- **Pause**: Suspend execution
- **Resume**: Continue paused execution
- **Cancel**: Stop execution entirely

For **Human Approval** nodes:
- **Approve**: Continue to the next step
- **Reject**: Stop or branch to rejection path

### Managing Workflows

**Clone a Workflow:**
1. From the Workflows list, click the menu (three dots) on a workflow
2. Select **Clone**
3. A copy is created with "(Copy)" appended to the name

**Publish a Workflow:**
1. Open the workflow in the builder
2. Click the menu (three dots) in the header
3. Select **Publish**
4. The workflow status changes to "Published"

**Retire a Workflow:**
1. Open a published workflow in the builder
2. Click the menu and select **Retire**
3. The workflow can no longer be executed

**Delete a Workflow:**
1. From the Workflows list, click the menu on a workflow
2. Select **Delete**
3. Confirm the deletion

---

## 5. Templates

Templates provide pre-configured starting points for projects.

### Template Library

Navigate to **Templates** (`/templates`) to browse available templates.

[Screenshot: Template library page]

**Page Features:**
- **Search**: Find templates by name or description
- **Category Pills**: Filter by category (e.g., "All", "Web Development", "API")
- **Template Count**: Badge showing total available templates

### Browsing Templates

Each template card shows:
- Template icon
- Name
- Category badge
- Description preview
- Usage count
- **Use Template** button

### Previewing a Template

Click any template card to open the preview dialog:

[Screenshot: Template preview dialog]

**Preview Contents:**
- Template name and icon
- Full description
- **Workflow Stages**: List of pre-configured stages with:
  - Sequence number
  - Stage name
  - Description
  - Approval required badge (if applicable)

### Creating a Project from a Template

1. Click **Use Template** on a template card or in the preview dialog
2. You're redirected to the project creation flow with the template pre-selected
3. Customize the project name and settings
4. Click **Create**

The new project will be populated with all stages and configurations from the template.

---

## 6. Artifacts

Artifacts are generated outputs from workflows and AI interactions.

### Viewing Artifacts

From a project detail page, click the **Artifacts** tab.

[Screenshot: Artifacts list showing pending approvals badge]

**Artifacts Tab Features:**
- **Pending Approvals Badge**: Shows count of artifacts awaiting review
- **Artifact List**: All generated artifacts with:
  - File name/title
  - Type (code, documentation, config, etc.)
  - Status (approved, pending, rejected)
  - Created date
  - Preview/download options

### Artifact Statuses

| Status | Description |
|--------|-------------|
| **Pending Approval** | Awaiting human review before use |
| **Approved** | Reviewed and accepted |
| **Rejected** | Reviewed and not accepted |

### Approving Artifacts

For artifacts with "Pending Approval" status:

1. Click the artifact to view its contents
2. Review the generated content
3. Click **Approve** to accept or **Reject** to decline
4. Provide feedback (optional) when rejecting

Approved artifacts can be:
- Referenced in chat conversations
- Downloaded for local use
- Included in project exports

---

## 7. Account & Settings

### Profile Settings

Navigate to **Profile** (`/profile`) to manage your account information.

[Screenshot: Profile settings page]

**Editable Fields:**
- **Display Name**: How your name appears to others
- **Timezone**: Your local timezone for date/time display
- **Bio**: Brief description about yourself

### User Settings

Navigate to **Settings** (`/settings`) for application preferences.

[Screenshot: User settings page]

**Available Settings:**

**Theme:**
- Light mode
- Dark mode
- System (follows OS preference)

**Notifications:**
- Email notifications toggle
- In-app notifications toggle
- Notification frequency preferences

### Notifications Center

Navigate to **Notifications** (`/notifications`) to view all notifications.

[Screenshot: Notifications page]

**Notification Types:**
- Project invitations
- Workflow completion alerts
- Approval requests
- System announcements

**Actions:**
- Mark as read/unread
- Delete notifications
- Filter by type

---

## 8. Tips & Troubleshooting

### Productivity Tips

1. **Use Templates**: Start projects from templates to save time on initial setup

2. **Organize with Workspaces**: Group related projects into workspaces for better organization

3. **Name Conversations**: Give your AI chats descriptive titles for easy reference

4. **Save Workflow Drafts**: Click Save frequently in the workflow builder to avoid losing work

5. **Use Keyboard Shortcuts**:
   - `Ctrl/Cmd + S`: Save (in applicable contexts)
   - `Escape`: Close modals and panels

6. **Reference Artifacts in Chat**: Click artifacts in the context sidebar to give the AI specific context

### Common Issues and Solutions

**Issue: Workflow won't save**
- Ensure all nodes are properly connected
- Check for validation errors highlighted in red
- Verify you have edit permissions for the workflow

**Issue: AI chat isn't responding**
- Check your internet connection
- Refresh the page and try again
- If using a project agent, verify it's properly configured

**Issue: Can't see a project**
- Verify you have access to the project
- Check with the project owner about your role
- Ensure you're in the correct workspace

**Issue: Template not appearing**
- Templates must be marked as public to appear in the library
- Try clearing the search filter
- Refresh the page

**Issue: Workflow execution stuck**
- Check if it's waiting at a Human Approval node
- Look for errors in the execution log
- Try canceling and re-running the workflow

**Issue: Artifacts not generating**
- Verify the workflow completed successfully
- Check the workflow's artifact output configuration
- Ensure sufficient permissions exist

### Getting Help

If you encounter issues not covered here:
1. Check with your administrator
2. Review the Admin Guide for system configuration
3. Report issues at https://github.com/maraichr/GateHouse/issues

---

*GateHouse User Guide v1.0*
