# INQULA Connect
Technical documentation for connecting your organisation, users and AI tools to INQULA.
INQULA is designed as an AI-first platform. In addition to the INQULA Portal, authorised users can interact with INQULA through supported AI assistants using the Model Context Protocol (MCP).
## Identity and User Access
INQULA supports two identity models.
### Microsoft Entra ID
For organisations using Microsoft Entra ID.
Users sign in with their existing work accounts, and INQULA can synchronise users from the organisation's directory.
User synchronisation can include all relevant users or be limited to selected Entra ID groups.
## INQULA External ID
For organisations that do not use Microsoft Entra ID, or where a direct directory integration is not required.
INQULA provides authentication through Microsoft Entra External ID.
## AI and Model Context Protocol (MCP)
INQULA exposes supported functionality to AI assistants through the Model Context Protocol (MCP).
This allows authorised users to work with INQULA from supported AI tools while INQULA continues to enforce the user's access and permissions.
### How Access Works
The AI assistant does not replace INQULA's access control.
Each request is authenticated and evaluated against the user's INQULA permissions. The user can only access functionality and data the user is authorised to access in INQULA.
Organisations can also control whether use of the INQULA MCP application is permitted.
## INQULA Skills
INQULA Skills provide ready-made instructions for using AI assistants together with INQULA's MCP tools for specific tasks and workflows.
Available skills are published in the skills directory.
## Repository Structure
```INQULA-connect/
├── README.md
├── SECURITY.md
├── identity/
│   └── entra_id.md
├── ai/
│   ├── copilot.md
│   └── claude.md
└── skills/
    └── README.md```
