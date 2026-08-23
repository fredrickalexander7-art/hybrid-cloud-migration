# Incident Writeup — Azure Authentication Blocked by Security Defaults

## Situation

While building the Azure half of this project, the first `terraform plan` against Azure failed immediately with:

```
Error: building account: could not acquire access token to parse claims: running Azure CLI: exit status 1:
ERROR: AADSTS530035: Access has been blocked by security defaults.
```

Terraform was authenticating using a personal Azure CLI login (device-code flow), the same approach that had already worked fine for exploring the Azure Portal and running basic CLI commands. The error indicated that Azure's **Security Defaults** policy was blocking a Microsoft Graph API call the Terraform provider makes internally as part of its own authentication setup — a call that has nothing to do with the actual resources being created, but that the provider performs regardless.

## Task

Get Terraform authenticated against Azure in a way that would actually work, without simply disabling a security control to make an error go away.

## Action

1. **First hypothesis: incomplete MFA setup.** Attempted to complete Azure MFA configuration, since Security Defaults enforces MFA and the account's MFA setup had been left unfinished earlier in the project. This led to three separate failures across three different Microsoft login/security pages: the standalone self-service MFA page rejected the account as a "personal account" even when the correct email was used, and a second attempt through the Entra admin center directly failed with a generic `AADSTS90123: access_denied` error. None of these attempts resolved the underlying issue, and continuing to chase MFA setup through inconsistent Microsoft login flows was clearly not converging on a fix.

2. **Reframed the problem.** Rather than continuing to fight the personal account's authentication, stepped back and asked: should Terraform even be authenticating as *me* in the first place? On the AWS side of this same project, a dedicated IAM user (`falex-admin`) had already been created specifically so that automation and daily work never used the AWS root account. The same principle applies to Azure — a Terraform automation tool authenticating as a *personal* identity is the wrong pattern regardless of whether Security Defaults happens to allow it.

3. **Created a service principal.** Registered a dedicated Azure AD application (`terraform-sp`) via App Registrations, generated a client secret, and configured Terraform to authenticate using client-credentials flow (`ARM_CLIENT_ID` / `ARM_CLIENT_SECRET` / `ARM_TENANT_ID` / `ARM_SUBSCRIPTION_ID`) instead of a personal login. This is a fundamentally different, non-interactive OAuth flow that isn't subject to the same Security Defaults restriction that blocks device-code Graph access.

4. **Granted the service principal Contributor access on the subscription** — but the "Contributor" role wasn't visible in the default role-assignment search in the Azure Portal. Investigated and found it was filtered under a separate **"Privileged administrator roles"** tab rather than the general role list, since Contributor is high-privilege enough that Azure categorizes it separately. Selected it from the correct tab and assigned it successfully.

5. **First Terraform run with the service principal still failed** — a new error, `AADSTS7000215: Invalid client secret provided`, which explicitly stated the issue: *"Ensure the secret being sent in the request is the client secret value, not the client secret ID."* The wrong value had been copied from Azure's Certificates & Secrets table — "Secret ID" and "Value" are two different columns, and the wrong one had been saved. Generated a fresh secret and copied the correct column.

6. **Discovered a second, unrelated problem while troubleshooting.** The personal Azure CLI login had also stopped working (`No subscriptions found`), traced back to an earlier `az logout` (run during an earlier failed fix attempt) that had never been followed by a successful re-login. Resolved with `az account clear` followed by a fresh `az login`.

7. **Verified the fix independently before trusting Terraform again.** Rather than immediately re-running `terraform plan` and hoping, tested the service principal's credentials directly with a raw `curl` request against Azure's OAuth token endpoint (`client_credentials` grant). Getting back a valid access token confirmed the credentials were correct *before* re-introducing Terraform as a variable, isolating whether any remaining failure would be an auth problem or something else.

8. **`terraform plan` succeeded.** Applied cleanly — resource group, VNet, and subnets created using the service principal identity.

## Root cause

Terraform's Azure provider performs an internal Microsoft Graph API call as part of authenticating, and Azure's Security Defaults policy blocks that specific call for non-interactive (device-code) personal logins. The correct fix wasn't to weaken Security Defaults, but to stop authenticating Terraform as a personal identity at all — the same least-privilege automation principle already in use on the AWS side of this project, just implemented with Azure's identity model (service principal + RBAC role) instead of AWS's (IAM user + policy).

## Fix

Authenticate Terraform via a dedicated service principal (`terraform-sp`) with a scoped Contributor role assignment on the subscription, using client-credentials authentication instead of a personal Azure CLI login.

## Prevention / recommendations

- Set up automation identities (service principals / IAM users) *before* starting infrastructure-as-code work on a new cloud account, rather than defaulting to a personal login and discovering the problem mid-build.
- When an error message names the exact mistake (as `AADSTS7000215` did here — "value, not the ID"), read it carefully before assuming a deeper problem; not every error requires a redesign.
- Verify newly-created credentials independently (e.g. a direct OAuth request) before layering more tooling on top and re-testing — isolates whether a fix actually worked before adding more complexity back in.
- Document where a cloud provider's UI hides high-privilege options behind a separate filter/tab (as Azure does with "Privileged administrator roles") — easy to think an option doesn't exist when it's just filtered elsewhere.

---

## Additional troubleshooting notes (shorter incidents)

### AWS: Terraform version / GPG signing key / PATH conflict

The first `terraform plan` on AWS failed with a GPG "key expired" error while installing the AWS provider. Root cause: the VM's Terraform (v1.6.0, left over from an earlier project) was too old to recognize HashiCorp's current provider-signing key. Fixed by adding HashiCorp's official apt repository and installing the current Terraform version — but `terraform --version` still reported the old version afterward, because a stale binary at `/usr/local/bin/terraform` was earlier in the shell's `PATH` than the newly-installed one. Removed the stale binary and cleared the shell's command cache (`hash -r`) to resolve it. A four-step diagnostic chain (error → wrong-version hypothesis → fix attempt → new symptom → actual root cause) from a single initial error message.

### Both clouds: private subnets had no outbound internet access

On AWS, installing Docker on the private app server hung indefinitely with no error and no download progress. Diagnosed as a networking design gap, not a package-manager problem: the private subnet had no route to the internet at all (correct for *inbound* traffic, but also blocking necessary *outbound* traffic like package downloads). Fixed by adding a NAT Gateway and a private route table directing outbound traffic through it. Applied the same fix proactively on Azure *before* attempting any package installs there, based directly on the AWS lesson — Docker installed on the Azure app VM without any hang or delay on the first attempt.
