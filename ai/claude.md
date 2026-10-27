# Connecting Claude

Claude reaches Inqula over the Model Context Protocol (MCP). This page covers the whole setup, for
both login models.

## Connecting when your organisation runs its own Entra ID

This is the common case: your users sign in to Microsoft with their work accounts, and Inqula
already syncs them from your directory. Nobody gets a new account, and your IT keeps control of
the whole lifecycle.

**Connecting an AI tool to Inqula is a decision for your IT department, not for individual
users.** The setup is built so that nobody can connect Claude to your Inqula data until an
administrator in your organisation has approved it — and IT can withdraw that approval at any time.

### Before you start

Three things must be in place. Your Inqula contact confirms all of them:

| | Why |
|---|---|
| **The AI module is enabled** for your organisation | Without it every request is rejected, in the portal and in Claude alike |
| **Your users are synced** into Inqula | An Entra user with no Inqula account is refused — authentication alone is not access |
| **Your tenant is registered** with Inqula | Each organisation is admitted explicitly, by configuration. You are not let in by merely having an Entra ID |

To register your tenant, Inqula needs your **Directory (tenant) ID**, found on the overview page
of Microsoft Entra admin center.

You then receive your connector address. It contains a random route id assigned to your
organisation, not your organisation's name, so the address does not reveal that you use Inqula:

```
https://ai.inqula.com/t/<your-route-id>/mcp
```

### Step 1 — Your IT administrator approves the app

The **INQULA MCP** application must be approved in your Entra directory. This happens once, for
the whole organisation, and must be done by a Global Administrator, Cloud Application
Administrator or Application Administrator.

Approve it **in advance**, before any user tries to connect, by opening:

```
https://login.microsoftonline.com/<your-tenant-id>/adminconsent?client_id=4bd6bd82-0630-4d49-ab73-f2c3b69396c9
```

Sign in as the administrator and accept. The app then appears under **Enterprise applications**
as *INQULA MCP*.

We recommend this route rather than approving on a user's first sign-in: it makes the approval a
deliberate decision by IT, not a prompt someone clicks through. If user consent is allowed in your
tenant, a user could otherwise approve the app for themselves.

The app asks for:

| Permission | What it means |
|---|---|
| `openid`, `profile`, `offline_access` | Sign in and stay signed in — the standard set |
| `access_as_user` | Work in Inqula **as the signed-in user**, with exactly that person's rights |

It asks for nothing else. It cannot read your mail, your files or your directory.

The approval is recorded in your Entra audit log — who approved it, and when. That record is
useful documentation for your own supplier and ICT third-party follow-up (for example under DORA).

### Step 2 — Decide who may use it (recommended)

By default every user in your directory who also has an Inqula account can connect once the app
is approved. To limit it to named people:

1. In **Microsoft Entra admin center → Enterprise applications → INQULA MCP → Properties**, set
   **Assignment required?** to **Yes** and save.
2. Under **Users and groups**, add the users or a group — for example *Inqula AI users*.

Users who are not assigned are refused at sign-in, even if they have access to the Inqula portal
in the browser.

Your existing **Conditional Access** policies — MFA, compliant devices, location rules — apply to
this sign-in automatically. Nothing needs to be configured for Inqula.

### Step 3 — Add the connector in Claude

In Claude: **Settings → Customize → Connectors → Add custom connector**. On a Team or Enterprise
plan, an owner adds the connector once for the whole organisation, and members only need to
connect.

| Field | Value |
|---|---|
| URL | `https://ai.inqula.com/t/<your-route-id>/mcp` |
| Authentication | *Always required* — Claude detects this |
| OAuth client | *Use your own OAuth client* — Claude detects this too |
| Client ID | `4bd6bd82-0630-4d49-ab73-f2c3b69396c9` |
| Client secret | **leave empty** |

Two fields decide whether this works.

**The URL must contain `/t/<your-route-id>`, exactly as you received it.** Without that segment you are sent to Inqula's
own customer directory, where your work account does not exist, and the sign-in fails with a
generic authorization error.

**The client secret field must stay empty.** The application is registered as a public client
using PKCE, so it has no secret at all — entering anything there makes the sign-in fail. Nothing
confidential is handed out, and nothing is shared between customers.

A client ID is still required because Microsoft Entra does not support automatic client
registration. Claude detects this and points you at the same option. The client ID is public
information: it names the application, it grants nothing.

### Step 4 — Sign in

Each user signs in with their own work account. No new passwords, no separate accounts — your IT
keeps the whole lifecycle.

From then on Claude can only see and change what that person could already see and change in the
portal: same company, same organisational unit, same permissions. A user who cannot delete a
client in the portal cannot delete one through Claude either.

A user who authenticates successfully but has never been set up in Inqula is refused. That is
intentional: signing in to Microsoft is not the same as having an Inqula account.

### Who controls what

| Layer | Controlled by | Decides |
|---|---|---|
| Entra ID | **Your IT** | Whether the connection is allowed at all, and who may sign in |
| Inqula portal | **Inqula, per your roles** | What each user can see and change — same rights as in the browser. And even overall accesss to use external AI-tools |
| Claude | **Your organisation** | Which AI provider you use, and on what terms. Anthropic is your own supplier, and is assessed as part of your own supplier follow-up |

### Revoking access

IT can stop all use immediately, without involving Inqula:

- **Pause it:** Enterprise applications → INQULA MCP → Properties → **Enabled for users to sign
  in? = No**.
- **Remove a single user:** remove them from the assigned group (requires Step 2).
- **Remove it entirely:** delete the INQULA MCP enterprise application. A new admin approval is
  then needed to reconnect.

Tell your Inqula contact if you also want the AI module switched off on the Inqula side.

---

## If Inqula manages your login instead

Some customers do not federate their own directory and instead get accounts in Inqula's customer
directory. The setup is the same as Steps 3 and 4 above, except the address has no organisation
segment:

```
https://ai.inqula.com/mcp
```

Here there is no Entra approval on your side. Access is controlled by Inqula enabling the AI
module and the user's role in the portal — agree with your Inqula contact who should have it.

Your Inqula contact tells you which of the two applies to you.

---

## What Claude can do once connected

The tools come from Inqula's own catalogue, so anything the portal can do, Claude can do — reading
risks and deviations, creating measures, checking compliance status, following up periodic tasks.

**Claude never gets more access than the person using it.** Every call is checked against that
user's permissions, in the same layer the portal uses. A user who cannot delete a client in the
portal cannot delete one through Claude either.

## Skills

Ready-made skills for Claude live in [`../skills/claude/`](../skills/claude/). Each one states
which tools it needs.

## Troubleshooting

**"Need admin approval"** — Step 1 has not been done. Ask your IT administrator to open the
approval link above.

**"User is not assigned to a role for the application" (AADSTS50105)** — Assignment is required
(Step 2) and this user is not in the assigned group. Add them in Entra.

**"Authorization failed"** — nearly always the address. Check that it contains
`/t/<your-route-id>` and that the client ID is exactly the one above. If you entered a client
secret, remove it: the app does not use one, and supplying one makes the sign-in fail.

**"Method Not Allowed" in a browser** — expected. The address is not a web page; it is an endpoint
Claude talks to. This message means the server is reachable.

**Claude connects, but no tools appear** — the AI module is probably not enabled for your
organisation, or your user lacks permission to use it. Both are settings in Inqula, not in Claude.

**A user cannot sign in although the others can** — they are most likely not synced into Inqula,
or not assigned to the app in Entra (Step 2).

---

Questions: support@inqula.com
