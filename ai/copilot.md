# Connecting Microsoft Copilot

Copilot reaches Inqula over the Model Context Protocol (MCP), on the same server Claude uses. The
setup runs through **Copilot Studio**, and the agent you build there can be published to Microsoft
365 Copilot for your users.

This path is more involved than Claude's, and honestly so: two of the steps need Inqula to act,
and one needs capacity you buy from Microsoft. All three are listed below rather than discovered
halfway through.

## Before you start

Four things must be in place.

| | Why |
|---|---|
| **The AI module is enabled** for your organisation | Without it every request is rejected, in the portal and in Copilot alike |
| **Your users are synced** into Inqula | An Entra user with no Inqula account is refused — authentication alone is not access |
| **Your tenant is registered** with Inqula | Each organisation is admitted explicitly, by configuration |
| **Your environment has Copilot Studio credits** | Microsoft bills running an agent separately. Without assigned credits the agent can be built but not run |

The first three are confirmed by your Inqula contact. The fourth is yours: credits are purchased
at tenant level in the Power Platform admin center and must then be **assigned to the specific
environment**. Having credits in the tenant but none on the environment produces the same error as
having none at all.

You also receive from Inqula:

- your connector address, containing a random route id assigned to your organisation (not your
  organisation's name, so the address does not reveal that you use Inqula):
  `https://ai.inqula.com/t/<your-route-id>/mcp`
- a **client secret** for Copilot

## Step 1 — Your IT administrator approves the app

The **INQULA MCP** application must be approved in your Entra directory. This happens once, for
the whole organisation.

Most organisations let it happen on first sign-in. If your tenant blocks user consent — many do —
an administrator can approve it in advance:

```
https://login.microsoftonline.com/<your-tenant-id>/adminconsent?client_id=4bd6bd82-0630-4d49-ab73-f2c3b69396c9
```

The app asks for `openid`, `profile` and `access_as_user` — sign in, stay signed in, and work in
Inqula **as the signed-in user**. It asks for nothing else. It cannot read your mail, your files
or your directory.

## Step 2 — Add the MCP server in Copilot Studio

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

## Step 3 — Send the redirect URL to Inqula

When the server is created, Copilot Studio shows a **redirect URL**. It looks like this:

```
https://global.consent.azure-apim.net/redirect/cr61c-5f<your-server-name>-5f<hash>
```

**Send it to your Inqula contact.** It has to be registered against the application before anyone
can sign in, and only Inqula can do that.

This URL is unique to your connector — it encodes the server name you chose and a hash. It is
therefore different for every customer, and a new one is needed if you recreate the connector
under a different name. Claude needs no equivalent step; this one is specific to how Copilot
Studio handles OAuth.

If you skip this, sign-in fails with `AADSTS50011: The redirect URI ... does not match the
redirect URIs configured for the application`. The error text contains the exact URL that needs
registering, so it can simply be forwarded.

## Step 4 — Create the connection and add the tool

1. Select **Create a new connection** and sign in with your own work account
2. Approve the consent screen if shown
3. Select **Add to agent**

The tool then appears under **Tools** in the agent.

## Step 5 — Publish to your users

**Publish** the agent, then **Channels → Teams and Microsoft 365 Copilot** to make it available in
Microsoft 365 Copilot. Share it with the users or groups who should have it.

Each user creates their own connection the first time they use it, and signs in as themselves.

## What Copilot can do once connected

The tools come from Inqula's own catalogue, so anything the portal can do, Copilot can do —
reading risks and deviations, creating measures, checking compliance status, following up periodic
tasks.

**Copilot never gets more access than the person using it.** Every call is checked against that
user's permissions, in the same layer the portal uses — same company, same organisational unit,
same rights. A user who cannot delete a client in the portal cannot delete one through Copilot
either.

## Troubleshooting

**"Could not create MCP server. Try again."** — most often a name collision with an existing
connector, including one left behind by a failed attempt. Change the server name. The real error
is visible in the browser's network panel: a `409 Conflict` confirms it.

**`AADSTS50011` redirect URI mismatch** — step 3 has not been done, or the connector was recreated
and now has a different redirect URL. Forward the URL from the error text to Inqula.

**`AADSTS65001` consent required** — an administrator has not approved the application. See
step 1.

**"You need credits to continue" / `EnforcementUsageCredits`** — the environment has no Copilot
Studio credits assigned. This is a Microsoft licensing matter, not an Inqula one. Assign capacity
to the environment in the Power Platform admin center.

**Copilot connects, but no tools appear** — the AI module is probably not enabled for your
organisation, or your user lacks permission to use it. Both are settings in Inqula.

**A user cannot sign in although the others can** — they are most likely not synced into Inqula.
Authenticating with Entra is not the same as having an Inqula account.

## How this differs from Claude

Both use the same server, the same tools and the same permission model. The differences are
operational, and worth knowing before you choose:

| | Claude | Copilot |
|---|---|---|
| Client secret | None — public client with PKCE | Required, issued by Inqula |
| Redirect URL | Fixed, already registered | Unique per customer, Inqula must register it |
| Running cost | None beyond your Claude licence | Copilot Studio credits, billed by Microsoft |

---

Questions: info@inqula.com
