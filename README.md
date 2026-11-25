# Phase-2-Terraform

### Step 16: OIDC sign-in

**Goal:** Let NovaLab employees sign in to an app with Entra ID using OpenID Connect, and inspect the ID token to understand what the app receives.

**What I did:**
1. Granted the Terraform service principal `Application.ReadWrite.OwnedBy`, so it can manage only the app registrations it owns.
2. Defined the **NovaLab Employee Portal** app registration and service principal in Terraform, through a pull request, plan and approved apply.
3. Assigned the app to **NovaLab Synced Users**, so only enabled employees can sign in.
4. Wrote a Python app that signs in with the Authorization Code flow and PKCE, then prints the ID token's claims.
5. Tested access: an assigned user signed in.

<img width="1019" height="797" alt="Screenshot 2026-11-24 at 23 29 15" src="https://github.com/user-attachments/assets/d2a3aed5-4fd0-4ca3-a065-47671a19bf1d" />

<img width="819" height="542" alt="Screenshot 2026-11-24 at 18 05 38" src="https://github.com/user-attachments/assets/5ffd75f5-8d9f-47fd-a7dc-7cb5a824be37" />

<img width="1399" height="959" alt="Screenshot 2026-11-24 at 23 29 45" src="https://github.com/user-attachments/assets/3f7b9dbb-e161-45e9-b84c-796a3f9d436f" />

<img width="1260" height="707" alt="Screenshot 2026-11-24 at 15 06 58" src="https://github.com/user-attachments/assets/05f7fc3e-3454-4016-b341-2ded5f0ba168" />

<img width="849" height="640" alt="Screenshot 2026-11-24 at 17 02 58" src="https://github.com/user-attachments/assets/23a7e9e4-bb1e-4894-969d-361558e3d51c" />

**What OIDC is:**
OpenID Connect answers "who is this user?" for an app. It's built on OAuth 2.0 and returns a signed **ID token** (a JWT) describing the user.

**The flow (Authorization Code + PKCE):**

```
1. The app creates a random secret (code verifier) and its hash (code challenge)
2. The browser goes to Entra with client_id, redirect_uri, scopes and the code challenge
3. Entra signs the user in (password, MFA, Conditional Access, assignment check)
4. Entra redirects to http://localhost with a one-time authorization code
5. The app exchanges the code plus the code verifier for tokens
6. Entra checks the verifier matches the challenge, then returns the ID token
```

PKCE means a stolen authorization code is useless without the verifier, which never left the app. That's why a **public client** (an app on the user's device) can sign users in safely **without a client secret**.

| `Need admin approval` at sign-in | The tenant doesn't allow users to consent to apps, so the `User.Read` scope needed an admin's approval | Granted admin consent for the tenant (portal, or `az ad app permission admin-consent`). This restriction protects against consent phishing |

<img width="734" height="466" alt="Screenshot 2026-11-24 at 16 48 10" src="https://github.com/user-attachments/assets/dcbf06d8-fab3-40de-9736-5113022a1f81" />

<img width="816" height="622" alt="Screenshot 2026-11-24 at 18 09 27" src="https://github.com/user-attachments/assets/a3e719d6-8ac0-4831-a61c-b6b2da212ccb" />

**The app registration:**

| Setting | Meaning |
|---|---|
| `sign_in_audience = "AzureADMyOrg"` | Only accounts in my tenant |
| `public_client` + `http://localhost` | Runs on the device; Entra returns the code to a local port |
| `group_membership_claims = ["SecurityGroup"]` | Put the user's security group IDs in the token |
| `optional_claims` `email` | Add the email to the ID token |
| `required_resource_access` `User.Read` | Request permission to read the signed-in user's profile |
| `app_role_assignment_required = true` | Only assigned users and groups can sign in |


<img width="716" height="606" alt="Screenshot 2026-11-24 at 23 24 23" src="https://github.com/user-attachments/assets/6cc19c94-092d-4b43-b491-058aeb0b830a" />

**App registration vs service principal:** the application is the global definition; the service principal (the "Enterprise application") is its instance in my tenant, where assignment and sign-in policies apply.

**Reading the ID token:**

| Claim | Meaning |
|---|---|
| `iss` | The issuer: my Entra tenant |
| `aud` | The audience: this app's client ID; apps must reject tokens issued for others |
| `oid` | The user's object ID |
| `tid` | The tenant ID |
| `groups` | Security group object IDs, not names, because IDs never change |
| `alg`, `kid` (header) | The signing algorithm and the key used; apps verify the signature with Entra's published keys |

<img width="1661" height="584" alt="Screenshot 2026-11-24 at 23 25 03" src="https://github.com/user-attachments/assets/c9c20ae9-f5cc-4552-8f26-1cc45b3f8dd9" />

**Least privilege decisions:**
- `Application.ReadWrite.OwnedBy` instead of `Application.ReadWrite.All`: the service principal can't modify apps it didn't create.
- The group assignment is done in the portal: automating it would require `AppRoleAssignment.ReadWrite.All`, which can grant any app any permission, including itself.

<img width="999" height="610" alt="Screenshot 2026-11-24 at 14 49 04" src="https://github.com/user-attachments/assets/95f39bbb-5246-4a81-82e4-98824046f79f" />

**Access test:**

| User | Result | Why |
|---|---|---|
| Sam Chen (enabled, in NovaLab Synced Users) | Signed in | Assigned through the group |

<img width="805" height="611" alt="Screenshot 2026-11-24 at 18 03 15" src="https://github.com/user-attachments/assets/9b89d6a3-3df0-43d2-bea4-3bd4353d6dee" />

<img width="770" height="536" alt="Screenshot 2026-11-24 at 18 06 26" src="https://github.com/user-attachments/assets/b007449a-fa1b-4289-bda6-ab60fb580b9a" />

<img width="865" height="273" alt="Screenshot 2026-11-24 at 19 15 56" src="https://github.com/user-attachments/assets/839c8b2b-eb59-4d57-816a-1c8fcd5593a8" />

**Key terms:**
- **ID token:** a signed JWT that tells an app who the user is.
- **Public client:** an app that can't keep a secret (desktop, mobile, single-page app).
- **PKCE:** proof that the app redeeming the code is the one that requested it.
- **Consent:** a user or admin approving the permissions an app asks for.
- **Claims:** the fields inside a token.

**Commands used (in order):**

1. *(Entra: granted `Application.ReadWrite.OwnedBy` with admin consent.)* Create a branch.
   ```bash
   git checkout main
   git pull
   git checkout -b oidc-portal-app
   ```

2. *(VS Code: created `entra/apps.tf`, added the output.)* Format, commit, push, open a pull request and wait for checks and the plan.
   ```bash
   terraform fmt -recursive
   git add .
   git commit -m "Step 16: NovaLab Employee Portal app registration (OIDC)"
   git push -u origin oidc-portal-app
   gh pr create --fill
   gh pr checks --watch
   ```

3. Merge; then *(HCP Terraform: confirmed and applied the run)*.
   ```bash
   gh pr merge --merge --delete-branch
   git pull
   ```

4. *(Entra: assigned NovaLab Synced Users to the app; reset Sam Chen's password.)* Create a Python environment and install MSAL.
   ```bash
   mkdir -p apps/oidc-signin
   cd apps/oidc-signin
   python3 -m venv .venv
   source .venv/bin/activate
   pip install msal
   ```

5. Read the app's client ID from Terraform's outputs, and the tenant ID from the Azure CLI.
   ```bash
   cd ../../entra
   terraform output -raw employee_portal_client_id
   az account show --query tenantId -o tsv
   cd ../apps/oidc-signin
   ```

6. Set the IDs and run the sign-in.
   ```bash
   export NOVALAB_CLIENT_ID="<client ID>"
   export NOVALAB_TENANT_ID="<tenant ID>"
   python3 signin.py
   ```

7. Translate a group ID from the token into its name.
   ```bash
   az ad group show --group <group ID> --query displayName -o tsv
   ```
