# Security and data processing

This page is written for the person in your organisation who approves **INQULA MCP** in Microsoft
Entra ID: your IT administrator, security officer or IT supplier. It answers the questions we
usually get before admin consent. The step-by-step setup is in [claude.md](claude.md) and
[copilot.md](copilot.md).

## The short version

- The app lets an AI assistant work in the Inqula portal **as the signed-in user, with that
  user's rights and nothing more**. It cannot read mail, files or your directory.
- The Inqula MCP server **stores nothing**. Your data stays in the Inqula portal, where it
  already is.
- The AI assistant is **your own**. Your organisation chooses the AI provider and has its own
  agreement with it. Inqula is not a party to that agreement.
- Every change made through an AI assistant is logged in the portal's audit log with the user's
  name and the channel it came through.
- Your IT can stop all use at any time, without involving Inqula.

## The application

| | |
|---|---|
| Name in Entra | INQULA MCP |
| Owner | Inqula AS. Registered as a multi-tenant application in Inqula's own Entra ID, so each customer consents to the same app without a new registration |
| Client type | **Claude:** public client with PKCE, no client secret. **Copilot Studio:** requires a client secret, issued by Inqula to your organisation. In both cases the client ID is public and grants nothing on its own |
| Connector address | `https://ai.inqula.com/t/<your-organisation>/mcp` |

## Permissions

| Permission | What it gives |
|---|---|
| `openid`, `profile`, `offline_access` | Standard sign-in, and staying signed in (`offline_access` is requested by Claude, not by Copilot) |
| `access_as_user` | Calls the Inqula portal **on behalf of the signed-in user**. The portal checks every call against that person's roles: same company, same organisational unit, same rights as in the browser |

`access_as_user` is a permission on Inqula's own API. It gives no access to Microsoft Graph, so
nothing in your Microsoft 365 tenant (mail, files, Teams, directory) is reachable through it.

## What the assistant can read and change

The assistant gets the tools the user is allowed to use in the portal, for example risks,
deviations (quality events), measures, controls, requirements and compliance status, vendors,
periodic tasks and workflow tasks.

**Yes, it can also create and change information**, but never beyond the user's own rights. A user
who cannot delete a client in the portal cannot delete one through the assistant either. The
assistant does not have its own identity or a service account; it always acts as a named person.

Three things must be true before anything works:

1. Your IT has given admin consent to the app (and, if you choose, assigned it to named users or
   a group).
2. The user exists in Inqula. Signing in to Microsoft is not the same as having an Inqula account.
3. Inqula has enabled the AI module for your organisation and registered your tenant. Each
   customer is admitted explicitly, by configuration.

### How tokens are checked

- Each customer gets its own address (`/t/<your-organisation>/mcp`). The MCP server accepts a
  token on that address only if it was **issued by your Entra tenant** (issuer and `tid` are both
  pinned). A valid token from another organisation is rejected.
- The MCP server accepts only tokens issued **for this application** (audience check). A token
  issued for another app in the same directory is rejected.
- Your Conditional Access policies (MFA, compliant device, location) apply to the sign-in as
  usual.

## Data flow and storage

```
Your user in your AI assistant
   │  HTTPS, user's own token
   ▼
Inqula MCP server (ai.inqula.com)           stateless, stores nothing
   │  forwards the same token
   ▼
Inqula portal API                           checks rights, writes audit log
   │
   ▼
Inqula portal database                      where your data already lives
```

The Inqula MCP server runs in Inqula's production environment in Microsoft Azure, next to the
portal. It holds no credentials of its own and keeps no copy of your data. Each request is
handled on its own, bound to the caller's token, so one user's identity never carries over to
another's. Like any web service, it writes technical operations logs for troubleshooting; these
are retained for 7 days.

## Who is responsible for what

| | Role | Agreement |
|---|---|---|
| **Inqula** | Data processor for your data in the Inqula portal, as today | Your existing agreement with Inqula. The MCP server does not change it |
| **Your AI provider** (for example Anthropic for Claude, Microsoft for Copilot) | Processes what your users fetch into the assistant | **Your own agreement** with that provider. Inqula is not a party |
| **Your organisation** | Chooses the AI provider, the plan and its settings | Your own supplier follow-up, for example under DORA or GDPR |

In practice this means:

- What happens to data **inside the AI assistant** (retention, whether it may be used for
  training, data location) is governed by **your** organisation's account and terms with the
  provider, not by Inqula.
- We recommend a business plan with central administration (for Claude: Team or Enterprise),
  where an owner adds the Inqula connector for the organisation. Personal accounts should not be
  connected to company data.
- Restricting the app to a group in Entra controls **who** can sign in. Which AI account they use
  is controlled in your AI provider's admin settings.

### When Inqula itself uses AI in your tenant

Inqula support can, where you have allowed it, work in your tenant through a separate support
route with its own AI assistant, for example when helping you with setup. That use runs under
**Inqula's** business agreement with the AI provider, and there the provider is Inqula's
subprocessor. Everything Inqula support does is marked as Inqula support in your audit log, so
you can see it.

## Logging

| Where | What you can see |
|---|---|
| **Your Entra ID** | Who gave admin consent and when (audit log). Every sign-in to INQULA MCP (sign-in log) |
| **Inqula audit log** | Creates, changes and deletes in the portal's records: who, when, which object, the values before and after, and the channel it came through. Sign-ins and Inqula support access are logged too |

Each change records the channel it came through:

| Channel | Meaning |
|---|---|
| `portal` | A person working in the browser |
| `chat` | The portal's built-in AI assistant |
| `mcp` | An external AI assistant, such as Claude or Copilot |
| `api` | A direct call to Inqula's API from outside |
| `system` | Inqula's own background jobs, such as user sync from your directory and integrations with your other systems |

Background jobs are logged under a named system actor, for example *system: Unicorn integration*,
never under a person. A record created by an integration therefore never appears as if one of your
employees registered it.

Pure reads are not logged one by one. The audit log shows what was **changed** through an AI
assistant, not everything it read. Reads of the audit log itself are logged.

Your tenant administrator can read the audit log in the portal, filtered by channel, so you can
see exactly what has been done through AI assistants.

## Revoking access

Your IT can stop all use immediately, without involving Inqula:

- **Pause it:** Enterprise applications → INQULA MCP → Properties → **Enabled for users to sign
  in?** = No.
- **Remove one user:** remove them from the assigned group (requires *Assignment required* = Yes).
- **Remove it entirely:** delete the INQULA MCP enterprise application. A new admin consent is
  then needed.

Inqula can also switch off the AI module for your organisation. Then every request is rejected,
whatever the state in Entra.

---

Questions: support@inqula.com
