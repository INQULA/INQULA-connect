# Microsoft Entra ID authentication and user provisioning for INQULA

This document is intended for the customer's IT administrator or technical contact. It describes the two supported identity models for accessing INQULA and what the customer must do in each model.

## Choose the identity model

INQULA can use either:

1. **Customer Microsoft Entra ID** — recommended when the customer has its own Microsoft Entra ID tenant and wants to manage INQULA users through its existing identity environment.
2. **INQULA External ID** — available when the customer does not have, or does not want to integrate, its own Microsoft Entra ID tenant. INQULA provisions the users in INQULA's shared Microsoft Entra External ID customer directory.

The identity model is configured per INQULA tenant.

---

# Option 1: Customer Microsoft Entra ID

With this model, users authenticate with their existing work account. The customer remains responsible for managing user identities in Microsoft Entra ID.

Two application registrations are involved:

- **INQULA sign-in application** — a multi-tenant application owned and operated by INQULA for interactive sign-in.
- **Customer synchronization application** — a single-tenant application created and controlled by the customer. INQULA uses it to read the users and, where applicable, groups required for synchronization through Microsoft Graph.

The synchronization application uses Microsoft Graph **application permissions** and does not require an interactive user.

## Select the synchronization scope

Agree with INQULA on one of these models.

### A. All users

Use this option when all relevant active users in the customer tenant should be synchronized to INQULA.

No dedicated INQULA group is required.

### B. Selected Entra ID groups

Use this option when the customer wants Entra ID to determine which users are made available to INQULA.

The customer creates or selects one or more groups and provides the group's **Object ID** to INQULA.

Example:

- `INQULA-Users`

A user is added to or removed from the agreed group in Microsoft Entra ID. INQULA then uses that membership as the synchronization scope.

> Group membership and application roles inside INQULA are separate concepts unless a specific group-to-role mapping has been agreed and implemented.

## Step 1: Record tenant information

In the Microsoft Entra admin center, record:

- Tenant ID / Directory ID
- Primary domain

## Step 2: Create the synchronization app

Go to **Identity > Applications > App registrations > New registration**.

Use:

| Setting | Value |
|---|---|
| Name | `INQULA - User Synchronization` |
| Supported account types | Accounts in this organizational directory only (single tenant) |
| Redirect URI | Leave blank |

After registration, record:

- Application (client) ID
- Directory (tenant) ID

The background synchronization application does not need an exposed API scope for the INQULA synchronization process described here.

## Step 3: Create a client secret

Under **Certificates & secrets**, create a client secret using an expiry period that complies with your organization's policy.

Copy the secret **Value** immediately. Do not send INQULA the Secret ID.

Store the credential in the customer's approved secrets-management solution and transmit it to INQULA through an agreed secure channel.

## Step 4: Configure Microsoft Graph permissions

Under **API permissions**, select:

**Microsoft Graph > Application permissions**

For user synchronization add:

- `User.Read.All`

For group-based synchronization also add:

- `Group.Read.All`

Grant admin consent for the customer tenant and verify that the required permissions show **Granted**.

The integration described here uses read permissions. INQULA does not require this synchronization application to create, edit or delete Entra ID users or groups.

## Step 5: Configure groups, when applicable

When group-based synchronization is selected:

1. Create or select the group to use for INQULA access.
2. Add the relevant users.
3. Open the group overview.
4. Copy its **Object ID**.
5. Send the group name and Object ID to INQULA.

Use direct membership unless another membership model has specifically been agreed with INQULA.

## Step 6: Send the configuration to INQULA

Send:

- Tenant ID / Directory ID
- Primary domain
- Application (client) ID
- Selected synchronization model: all users or selected groups
- Group name and Object ID, when applicable
- Name and email of the first INQULA administrator
- Technical contact

Send separately through an agreed secure channel:

- Client secret **Value**
- Client secret expiry date

Never put secrets in this GitHub repository.

## Ongoing administration

The customer's Entra ID remains the source for user identity and group membership.

Typical administration is therefore performed in Microsoft Entra ID:

- add a new employee in Entra ID;
- add or remove membership in the agreed INQULA group when group-based synchronization is used;
- disable the Entra account when the user must no longer be able to authenticate;
- maintain name and email information in Entra ID; and
- replace the integration credential before it expires.

---

# Option 2: INQULA External ID

Use this model when the customer does not have its own Microsoft Entra ID tenant, or when INQULA and the customer agree not to integrate the customer's directory.

INQULA uses a shared **Microsoft Entra External ID (CIAM)** directory for customer identities.

The customer does **not** need to:

- create an Entra tenant;
- create the INQULA synchronization app registration;
- configure Microsoft Graph application permissions for INQULA; or
- provide a client secret to INQULA.

## What INQULA does

INQULA:

1. provisions the agreed user with email address, name, INQULA role and company access;
2. configures the INQULA tenant to use **INQULA External ID** authentication;
3. maintains the External ID integration used by the INQULA Portal; and
4. binds the External ID identity to the pre-provisioned INQULA user when the user signs in.

The authentication mode applies to the INQULA tenant.

## What the customer must provide

For each user to be provisioned, provide:

- Name
- Email address
- Required INQULA role
- Company/access scope, where applicable

Also identify the customer's administrative contact who is authorized to request user additions, access changes and removals.

Do not send passwords to INQULA.

## First sign-in

With the current INQULA External ID configuration:

1. the user is provisioned by INQULA;
2. the user opens the INQULA Portal;
3. the user signs up/signs in using the provisioned email address;
4. the user sets a password as part of first use where required by the sign-up flow;
5. the configured External ID flow performs the required verification/MFA step; and
6. INQULA binds the External ID identity to the provisioned INQULA user.

## User changes and offboarding

Because the customer is not supplying an Entra directory in this model, user provisioning is handled through INQULA rather than an automatic customer-directory synchronization.

Contact INQULA through the agreed support/administrative channel when a user must be:

- added;
- assigned different INQULA access;
- removed; or
- otherwise changed in the External ID model.

---

# Which model should we use?

| Customer situation | Recommended model |
|---|---|
| Customer has Microsoft Entra ID and wants existing work accounts | Customer Microsoft Entra ID |
| Customer wants Entra groups to control synchronization scope | Customer Microsoft Entra ID with selected groups |
| Customer does not have its own Microsoft Entra ID | INQULA External ID |
| Customer does not want to establish a directory integration | INQULA External ID, subject to agreement with INQULA |

---

# Security and operational notes

## Customer Entra ID

- Grant only the Microsoft Graph permissions required for the agreed synchronization model.
- Do not commit client secrets, certificates or customer-specific credentials to GitHub.
- Renew credentials before expiry.
- Remove unused integration credentials and permissions when the integration is terminated.

## INQULA External ID

- The customer does not receive INQULA's External ID credentials or application administration access as part of normal onboarding.
- Never send user passwords to INQULA.
- User additions, access changes and removals must come through the agreed administrative process.
- INQULA application roles remain separate from authentication.

---

# Troubleshooting

## Customer Entra ID: no users are synchronized

Verify:

- `User.Read.All` is an **Application** permission;
- admin consent has been granted;
- Tenant ID and Application (client) ID are correct;
- the client secret is valid; and
- the user is included in the agreed synchronization scope.

## Customer Entra ID: group users are missing

Verify:

- `Group.Read.All` has admin consent;
- INQULA has the correct group Object ID;
- the user is included in the agreed group membership model; and
- the Entra ID user account is active.

## Customer Entra ID: `AADSTS7000215` / invalid client secret

Create a new client secret and provide the secret **Value**, not the Secret ID, through the agreed secure channel.

## External ID: user cannot start or complete sign-in

Confirm first that the email address has been provisioned for the correct INQULA tenant. Send the affected email address and the exact error message to INQULA support. Never include a password or authentication code.

---

# Information to include when contacting INQULA

For **Customer Entra ID** issues:

- tenant/company name;
- Application (client) ID;
- affected group Object ID, where applicable;
- affected user's email address; and
- exact error message.

For **INQULA External ID** issues:

- tenant/company name;
- affected user's email address; and
- exact error message.

Never include passwords, one-time codes or client secrets in an ordinary support message or GitHub issue.
