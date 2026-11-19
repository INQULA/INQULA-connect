# Connecting Microsoft Copilot

Copilot reaches Inqula over the Model Context Protocol (MCP), on the same server Claude uses. There
are two ways to set it up:

- **Option A — a custom federated connector in the Microsoft 365 admin center** (recommended).
  The Inqula tools appear directly in Microsoft 365 Copilot chat. It needs only Microsoft 365
  Copilot licences, and nothing has to be registered by Inqula along the way.
- **Option B — an agent in Copilot Studio.** Use this if you want to build your own agent around
  the Inqula tools. It needs Copilot Studio credits, and Inqula must register a redirect URL for
  your connector before anyone can sign in.

Both use the same application, the same tools and the same permission model.

## Before you start

| | Why |
|---|---|
| **The AI module is enabled** for your organisation | Without it every request is rejected, in the portal and in Copilot alike |
| **Your users are synced** into Inqula | An Entra user with no Inqula account is refused — authentication alone is not access |
| **Your tenant is registered** with Inqula | Each organisation is admitted explicitly, by configuration |
| **Microsoft licensing** | Option A: a Microsoft 365 Copilot licence for each user. Option B: Copilot Studio credits assigned to the environment |

The first three are confirmed by your Inqula contact. Licensing is yours.

You also receive from Inqula:

- your connector address, containing a random route id assigned to your organisation (not your
  organisation's name, so the address does not reveal that you use Inqula):
  `https://ai.inqula.com/t/<your-route-id>/mcp`
- a **client secret** for Copilot

The bare address `https://ai.inqula.com` is not an MCP server and answers 404. Always use the full
address with your route id.

## Step 1 — Your IT administrator approves the app

The **INQULA MCP** application must be approved in your Entra directory. This happens once, for
the whole organisation, and applies to both options.

Most organisations let it happen on first sign-in. If your tenant blocks user consent — many do —
an administrator can approve it in advance:

```
https://login.microsoftonline.com/<your-tenant-id>/adminconsent?client_id=4bd6bd82-0630-4d49-ab73-f2c3b69396c9
```

The app asks for `openid`, `profile`, `offline_access` and `access_as_user` — sign in, stay signed
in, and work in Inqula **as the signed-in user**. It asks for nothing else. It cannot read your
mail, your files or your directory.

---

# Option A — Microsoft 365 admin center (recommended)

Two steps, both done by an administrator in your own Microsoft 365 tenant: first register the
OAuth client in the Teams Developer Portal, then create the connector in the Microsoft 365 admin
center.

## A1 — Register the OAuth client in the Teams Developer Portal

Go to [dev.teams.microsoft.com](https://dev.teams.microsoft.com) → **Tools** → **OAuth client
registration** → **New OAuth connection**, signed in with an account in your own tenant.

| Field | Value |
|---|---|
| Registration name | `Inqula` (any name you will recognise) |
| Base URL | `https://ai.inqula.com` |
| Restrict usage by organization | **My organization only** |
| Restrict usage by Teams app | **Any Teams app** |
| Client ID | `4bd6bd82-0630-4d49-ab73-f2c3b69396c9` |
| Client secret | *provided by your Inqula contact* |
| Authorization endpoint | `https://login.microsoftonline.com/<your-tenant-id>/oauth2/v2.0/authorize` |
| Token endpoint | `https://login.microsoftonline.com/<your-tenant-id>/oauth2/v2.0/token` |
| Refresh endpoint | Same as the token endpoint |
| Scope | `api://4bd6bd82-0630-4d49-ab73-f2c3b69396c9/access_as_user,openid,profile,offline_access` |
| Enable Proof Key for Code Exchange (PKCE) | **On** |
| Client password authentication method | **Request body parameters** (the default) |

Two of these differ from Copilot Studio:

**Scopes are separated by commas**, not spaces. The form says so in small print.

**The refresh endpoint is the token endpoint.** Entra uses the same endpoint for both.

Select **Save**. The portal shows an **OAuth registration ID**. Copy it — you need it in the next
step.

## A2 — Create the connector in the Microsoft 365 admin center

In [admin.cloud.microsoft](https://admin.cloud.microsoft): **Copilot** → **Connectors** → the
**Gallery** tab → **Create a new connector** → **Connect to MCP server**.

| Field | Value |
|---|---|
| Display name | `Inqula` (this is what users see in Copilot) |
| Description | `Inqula GRC — risk, deviations, measures, controls and compliance` |
| MCP endpoint URL | `https://ai.inqula.com/t/<your-route-id>/mcp` |
| Developer name | `Inqula` |
| Website URL | `https://www.inqula.com/` |
| Privacy policy URL | `https://www.inqula.com/` |
| Terms of use URL | `https://www.inqula.com/` |
| Authentication type | **OAuth 2.0** |
| Reference ID | The OAuth registration ID from step A1 |

Then:

1. Select **Authorize**. A sign-in window opens; sign in with your work account. If nothing
   happens and the form says the window *was blocked by the browser*, allow pop-ups for
   `admin.cloud.microsoft` and select **Authorize** again.
2. When the reference ID shows a green tick, accept the **Notice** and select **Create**.

Two choices in the form cannot work, so avoid them:

**Not "DCR (OAuth 2.0)".** Dynamic Client Registration is not offered by Microsoft Entra.

**Not "Entra SSO".** It issues a token for a different audience than the Inqula server accepts.
Use **OAuth 2.0**.

The redirect URL this path uses (`https://teams.microsoft.com/api/platform/v1.0/oAuthRedirect`) is
the same for every customer and is already registered on the Inqula application. There is nothing
to send to Inqula.

## A3 — Roll out to your users

The new connector appears in the connector list with state **Ready**. Open it:

- Under **Staged rollout**, select **Change** and choose who gets it — start with yourself or a
  small group, then widen.
- Make sure **Allow information from this connection to appear in Copilot** is **On**.
- Under **Tools**, select **Sign in** to see the list of Inqula tools the connector exposes.

If your organisation restricts agents and plugins, also check **Agents** → **Settings**:
**Agent and plugin access** must allow apps built by your organisation, and **User access** must
include the users who should have Inqula.

Each user signs in as themselves the first time they use Inqula in Copilot chat.

---

# Option B — Copilot Studio

This path is more involved: one step needs Inqula to act, and running the agent needs capacity you
buy from Microsoft.

Credits are purchased at tenant level in the Power Platform admin center and must then be
**assigned to the specific environment**. Having credits in the tenant but none on the environment
produces the same error as having none at all.

## B1 — Add the MCP server in Copilot Studio

In Copilot Studio: open your agent → **Tools** → **Add a tool** → **New tool** → **Model Context
Protocol**.

| Field | Value |
|---|---|
| Server name | Anything, but **unique in the environment** — see below |
| Server description | `Inqula GRC — risk, deviations, measures, controls and compliance` |
| Server URL | `https://ai.inqula.com/t/<your-route-id>/mcp` |
| Authentication | **OAuth 2.0** |
| Configuration type | **Manual** |
| Client ID | `4bd6bd82-0630-4d49-ab73-f2c3b69396c9` |
| Client secret | *provided by your Inqula contact* |
| Authorization URL | `https://login.microsoftonline.com/<your-tenant-id>/oauth2/v2.0/authorize` |
| Token URL | `https://login.microsoftonline.com/<your-tenant-id>/oauth2/v2.0/token` |
| Refresh URL | `.` — a single period |
| Scopes | `api://4bd6bd82-0630-4d49-ab73-f2c3b69396c9/access_as_user openid profile` |

Four of these trip people up.

**The server name must be unique.** If a connector with that name already exists in the
environment — including one from an earlier failed attempt — creation fails with
*"Could not create MCP server. Try again."* and no further explanation. The underlying error is a
409 Conflict. A failed attempt can still have left the connector behind, so if you are retrying,
change the name.

**The refresh URL is a single period.** Not the token URL. This is Microsoft's own documented
value for this dialog.

**Configuration type must be Manual.** The two *Dynamic* options require OAuth Dynamic Client
Registration, which Microsoft Entra does not offer. They cannot work, regardless of what else you
fill in.

**Paste cleanly.** A stray tab or space in front of a URL makes it invalid, and the dialog will
not tell you. Select the whole field and replace it rather than pasting alongside existing text.

## B2 — Send the redirect URL to Inqula

When the server is created, Copilot Studio shows a **redirect URL**. It looks like this:

```
https://global.consent.azure-apim.net/redirect/cr61c-5f<your-server-name>-5f<hash>
```

**Send it to your Inqula contact.** It has to be registered against the application before anyone
can sign in, and only Inqula can do that.

This URL is unique to your connector — it encodes the server name you chose and a hash. It is
therefore different for every customer, and a new one is needed if you recreate the connector
under a different name. Option A and Claude need no equivalent step.

If you skip this, sign-in fails with `AADSTS50011: The redirect URI ... does not match the
redirect URIs configured for the application`. The error text contains the exact URL that needs
registering, so it can simply be forwarded.

## B3 — Create the connection and add the tool

1. Select **Create a new connection** and sign in with your own work account
2. Approve the consent screen if shown
3. Select **Add to agent**

The tool then appears under **Tools** in the agent.

## B4 — Publish to your users

**Publish** the agent, then **Channels → Teams and Microsoft 365 Copilot** to make it available in
Microsoft 365 Copilot. Share it with the users or groups who should have it.

Each user creates their own connection the first time they use it, and signs in as themselves.

---

## What Copilot can do once connected

The tools come from Inqula's own catalogue, so anything the portal can do, Copilot can do —
reading risks and deviations, creating measures, checking compliance status, following up periodic
tasks.

**Copilot never gets more access than the person using it.** Every call is checked against that
user's permissions, in the same layer the portal uses — same company, same organisational unit,
same rights. A user who cannot delete a client in the portal cannot delete one through Copilot
either.

## Troubleshooting

**"This URL isn't formatted correctly"** on the MCP endpoint URL — the field contains something
other than the bare address, often stray text or whitespace from pasting. Clear the field and
paste the address alone.

**Authorize does nothing / "was blocked by the browser"** (option A) — the browser blocked the
sign-in pop-up. Allow pop-ups for the admin center and try again.

**`AADSTS50011` redirect URI mismatch** — option A: contact Inqula, as the fixed redirect URL
should already be registered. Option B: step B2 has not been done, or the connector was recreated
and now has a different redirect URL. Forward the URL from the error text to Inqula.

**`AADSTS65001` consent required** — an administrator has not approved the application. See
step 1.

**"Could not create MCP server. Try again."** (option B) — most often a name collision with an
existing connector, including one left behind by a failed attempt. Change the server name. The
real error is visible in the browser's network panel: a `409 Conflict` confirms it.

**"You need credits to continue" / `EnforcementUsageCredits`** (option B) — the environment has no
Copilot Studio credits assigned. This is a Microsoft licensing matter, not an Inqula one. Assign
capacity to the environment in the Power Platform admin center.

**The connector is created, but users do not see it** (option A) — check the staged rollout on the
connector, and the agent settings described in step A3.

**Copilot connects, but no tools appear** — the AI module is probably not enabled for your
organisation, or your user lacks permission to use it. Both are settings in Inqula.

**A user cannot sign in although the others can** — they are most likely not synced into Inqula.
Authenticating with Entra is not the same as having an Inqula account.

## How the options differ

All three use the same server, the same tools and the same permission model. The differences are
operational:

| | Claude | Copilot — admin center (A) | Copilot — Copilot Studio (B) |
|---|---|---|---|
| Client secret | None — public client with PKCE | Required, issued by Inqula | Required, issued by Inqula |
| Redirect URL | Fixed, already registered | Fixed, already registered | Unique per customer, Inqula must register it |
| Running cost | None beyond your Claude licence | None beyond Microsoft 365 Copilot licences | Copilot Studio credits, billed by Microsoft |
| Where users use it | Claude | Microsoft 365 Copilot chat | The agent you publish |

---

Questions: info@inqula.com
