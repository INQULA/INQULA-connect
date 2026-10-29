# System-to-system integration with INQULA

This document is intended for the customer's IT administrator, security officer or IT supplier.
It describes how INQULA recommends that INQULA and the customer's own business systems connect
to each other, and why. It then describes how this is done with **Microsoft Entra ID** and with
**Google Cloud**, each with a short guide to getting started.

## The short version

- INQULA connects **as a system, not as a person**, and only with the access the customer grants
  it.
- **No password or secret is handed over by the customer.** Access is based on short-lived tokens
  issued or trusted by the customer's own identity platform.
- The customer **controls the access in its own environment**: in its identity platform and in the
  business system. Either one can stop the integration immediately, without involving INQULA.
- Every connection is logged in the customer's own environment, and INQULA logs its own
  background jobs under a named system actor, never as a person.
- Access granted by one customer cannot be used at another customer.

---

# Our recommendation

## Identity comes from the customer's identity platform

INQULA recommends that system-to-system integrations use **short-lived tokens tied to the
customer's own identity platform**, rather than passwords or API keys:

- there is no shared secret that can leak, be copied or be forgotten when it should be rotated;
- the customer's own security policies, logging and revocation apply, in the customer's own
  environment;
- the customer's IT keeps the same tooling and control it already uses for its other systems.

## Least privilege

- INQULA is given a **dedicated identity and access**, separate from any person.
- The access is limited to what the agreed use requires. Administrator access should not be used
  in production.
- Where the business system supports it, the access is also limited to the relevant parts of the
  data.

## APIs, message queues and event streams

The business system may expose an **API** or a **messaging service** such as a message queue or
event stream (for example Apache Kafka, Azure Event Hubs, Azure Service Bus or Google Cloud
Pub/Sub). The principle is the same:

| | API | Message queue / event stream |
|---|---|---|
| How the token is presented | In the request (`Authorization: Bearer`) | When connecting (for Kafka: SASL/OAUTHBEARER) |
| How access is limited | A dedicated role in the API | Permissions per topic, queue or subscription, typically read on agreed topics only |

Messaging services often use other network ports than HTTPS. Whether INQULA can reach the service,
and how (for example allow-listing or private connectivity), must be clarified early.

## Direction of the integration

The model works in both directions:

- **INQULA connects to the business system** with a token tied to the customer's identity
  platform.
- **The business system delivers to INQULA** with a token issued in the customer's own
  environment, and INQULA accepts it only if it comes from that customer and is intended for
  INQULA.

No password or key is exchanged in either direction.

## How customers are kept apart

- **The customer's own environment decides.** INQULA can only reach a system where the customer
  has granted INQULA access.
- **The target is fixed per customer.** INQULA always uses the identity platform and access that
  belong to the INQULA tenant the integration runs in. This is taken from INQULA's own tenant
  configuration, not entered per integration, so a misconfigured address cannot make INQULA use
  another customer's access.
- **Tokens are checked by the receiver.** The business system accepts only tokens issued in the
  customer's own environment, intended for itself, and for an identity it has explicitly granted
  access. INQULA does the same for deliveries to INQULA.
- **Verified by test.** Before production, INQULA verifies that access obtained for one customer
  is **rejected** by another customer's system, and records the result.

## INQULA's own credential

- INQULA proves its own identity with a **certificate** held in INQULA's Azure Key Vault. It is
  not a password and it is never sent to the customer. It is rotated before expiry.
- The credential is the same for all customers; what it can reach in each customer is decided by
  that customer. It gives no access to a customer who has not granted INQULA access, and each
  customer can cut it off on its own.
- INQULA's target is a federated credential bound to INQULA's managed identity in Azure, so that
  no secret or certificate needs to be stored at all.

## Logging and revocation

- The customer sees INQULA's access in its own identity platform's logs, and in the business
  system's logs as far as the business system logs it.
- INQULA's audit log records background jobs under a named system actor for the integration,
  never as a person.
- The customer can stop the integration at any time, in its identity platform or in the business
  system, without involving INQULA.

---

# Microsoft Entra ID

## How it works

```
INQULA (runs in INQULA's Azure environment)
   │ 1. requests a token from the CUSTOMER'S Entra ID
   ▼
Customer's Microsoft Entra ID          issues a short-lived token on behalf of the
   │                                   INQULA application in the customer's tenant
   │ 2. token
   ▼
Customer's business system             validates the token and gives INQULA only the
                                       access the customer has granted
```

INQULA uses the **INQULA application** that the customer has already approved for signing in to
the INQULA Portal (see [entra_id.md](entra_id.md)). In the customer's
directory the application has its own identity (service principal). That identity is what the
business system recognises as INQULA. The token is obtained with the OAuth 2.0 *client
credentials* flow; no user is involved.

Azure Event Hubs, Azure Service Bus and managed Kafka services such as Confluent Cloud accept
Entra ID tokens. A self-hosted Kafka cluster can be configured to accept them.

When the business system delivers to INQULA, it obtains a token from the customer's Entra ID, and
INQULA accepts it only if it was issued by that customer's tenant for INQULA.

## Logging

| Where | What you can see |
|---|---|
| **Your Entra ID** | Every token INQULA obtains: *Sign-in logs → Service principal sign-ins* |
| **Your business system** | INQULA's calls, as far as the business system logs them |
| **INQULA audit log** | Background jobs under a named system actor for the integration |

## Revoking access

| Action | Effect |
|---|---|
| Remove INQULA's access in the business system | Stops the integration. Sign-in to INQULA is unaffected |
| Remove INQULA's application role assignment (step 3 below) | Stops the integration where *Assignment required* is set. Sign-in is unaffected |
| Disable the INQULA enterprise application in Entra ID | Stops **both** the integration and your users' sign-in to INQULA |

Personal guest accounts of INQULA staff in your directory are separate from the application and
must be removed separately if you want to remove all INQULA access.

## How to get started

Do this in a test or sandbox environment first, then repeat in production.

**In Microsoft Entra ID**

1. Go to **Enterprise applications** and find the INQULA application. It is present if your users
   already sign in to INQULA with their work accounts.
2. Open it and note its **Object ID** (not *Application ID*). This is INQULA's identity in your
   directory.
3. If the business system (API or messaging service) defines an application role for
   integrations in Entra ID, assign it to the INQULA application. This cannot be done in the Entra
   admin center; use Microsoft Graph PowerShell:

   ```powershell
   Connect-MgGraph -Scopes "AppRoleAssignment.ReadWrite.All","Application.Read.All"

   $inqula = Get-MgServicePrincipal -Filter "appId eq '<INQULA application ID>'"
   $api    = Get-MgServicePrincipal -Filter "appId eq '<business system API application ID>'"
   $role   = $api.AppRoles | Where-Object { $_.Value -eq '<application role value>' }

   New-MgServicePrincipalAppRoleAssignment -ServicePrincipalId $inqula.Id `
     -PrincipalId $inqula.Id -ResourceId $api.Id -AppRoleId $role.Id
   ```

   Consider setting **Assignment required? = Yes** on the business system's enterprise
   application, so that only applications that have been explicitly granted a role can obtain a
   token for it.

**In the business system**

4. Grant INQULA's identity from step 2 access to the business system, with a dedicated,
   least-privilege role as described above: in an API, a dedicated role; in a messaging service,
   permissions on the agreed topics or queues only. How this is done depends on the business
   system; consult its documentation or supplier.
5. For a messaging service, agree with INQULA how the service is reached over the network.

**Verify**

6. INQULA runs a health check that obtains a token from your Entra ID and makes one harmless
   read-only call or connection:

   | Result | Meaning |
   |---|---|
   | Entra ID refuses to issue a token | Consent or application role is missing (step 3) |
   | The business system rejects the token | INQULA's identity has not been granted access (step 4) |
   | The business system refuses the call | The granted access is too narrow for the agreed use (step 4) |
   | Success | The integration works |

7. When the test environment works, repeat steps 1–6 in production.

---

# Google Cloud

> Available on request. The setup is agreed with INQULA for each customer.

## How it works

Google Cloud uses **Workload Identity Federation**: the customer's Google Cloud project is set up
to trust INQULA's own identity, and exchanges it for a short-lived Google token. No service account
key is created or handed over, in line with Google's own recommendation.

```
INQULA (runs in INQULA's Azure environment)
   │ 1. presents INQULA's own identity token
   ▼
Customer's Google Cloud project          a workload identity pool trusts INQULA's identity
(Workload Identity Federation)           and issues a short-lived Google token for a
   │                                     dedicated service account in the customer's project
   │ 2. Google token
   ▼
Customer's business system on Google     gives that service account only the access granted
Cloud (API, Pub/Sub, ...)                in the customer's IAM
```

Access is controlled with **IAM roles** on the dedicated service account, for example read access
to one Pub/Sub subscription only.

When the business system delivers to INQULA, a service account in the customer's project obtains
a Google-signed identity token intended for INQULA. INQULA accepts it only if it is issued by
Google, intended for INQULA, and from the service account registered for that customer.

## Logging and revocation

- **Logging:** Cloud Audit Logs in the customer's project show the token exchanges and the
  service account's use.
- **Revocation:** remove INQULA's permission to use the service account, disable the workload
  identity provider or pool, or disable the service account. Each stops the integration
  immediately. Sign-in to INQULA is unaffected.

## How to get started

Do this in a test project first, then repeat in production.

1. Create a **workload identity pool** and a provider that trusts INQULA's identity, using the
   issuer and identity values INQULA provides. Restrict the provider to INQULA's identity only.
2. Create a **dedicated service account** for INQULA in the project.
3. Allow only INQULA's federated identity to use that service account
   (*Workload Identity User*).
4. Grant the service account the **least-privilege IAM roles** needed for the agreed use, on the
   specific resources only (for example one Pub/Sub subscription). Do not grant project-wide
   roles.
5. For a messaging service or private API, agree with INQULA how it is reached over the network.
6. INQULA runs a health check with one harmless read-only call or connection, and you repeat the
   setup in production when it works.

---

# Business systems that do not accept tokens

Some business systems only support their own users or API keys. INQULA then uses a dedicated
service account or key in that system, stored in INQULA's Azure Key Vault separately for each
customer, and limited to the access the integration needs. Repeated rejected logins stop further
attempts until the next day, so INQULA never locks the account through retries.

INQULA moves such integrations to a token-based model when the business system supports it.

---

Never include passwords, keys or tokens in an ordinary support message or GitHub issue.

Questions: support@inqula.com
