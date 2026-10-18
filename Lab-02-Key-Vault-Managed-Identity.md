# Lab 2 — Key Vault, managed identity, RBAC and network security

**Time:** 60–90 minutes. **Status:** planned, not deployed or live-tested. **Budget:** near-zero Key Vault operations, subject to existing Defender coverage; no Private Endpoint or compute in the core exercise. The series goal remains less than USD 4 additional Azure spend, not USD 4 per lab.

Reuse the actual Lab 1 names: `rg-sc500-lab1`, `sc500-mi`, `SC500-Security-Team`, `sc500-user1`. The supplied overview uses `rg-sc500-lab`; substitute `rg-sc500-lab1` throughout. Preserve Lab 1 resources at its cleanup checkpoint. Use a new isolated vault, leaving the existing `myvault3` untouched.

## What this lab proves

```text
Admin -- Secrets Officer at vault --> create/read dummy secret
Test user -- RG Reader -----------> inspect vault resource; cannot read value
Test user -- temporary Secrets User at vault --> read value; cannot write
sc500-mi -- Secrets User at vault -> authorized for future workload use
Client public IPv4 -- firewall allow --> data-plane request can reach vault
Same identity, IP rule removed ------> data-plane request blocked
```

The managed identity assignment is configuration evidence. A real identity-based secret retrieval needs a supported workload with `sc500-mi` attached, appropriate networking and a token for Key Vault. Local Azure CLI and ordinary Cloud Shell sessions run as their signed-in caller; neither automatically becomes this user-assigned identity. This core lab validates live secret access as users and prepares the workload pattern for a later lab.

## 1. Readiness and cost checkpoint (10 minutes)

Complete Lab 1's user/group/RBAC/identity steps first. Its optional PIM section is not required. Use the local Windows CLI for network tests, because your machine's public egress is the address being allowed. Portal secret operations also originate from your browser's network.

Before creating a vault, inspect Subscription 1 > Defender for Cloud > Environment settings > Defender plans > Key Vault coverage, effective plan, trial expiry and current pricing. The previous review found **KeyVaults = Standard / PerKeyVault** at subscription scope. A new vault may therefore incur workload-protection charges beyond secret operations. Do not assume free credit or trial coverage. If that pricing cannot fit the remaining series budget, stop before vault creation and keep this as a walkthrough until coverage/cost is resolved. Do not switch off subscription-wide protection to make this lab cheap.

Check [Key Vault pricing](https://azure.microsoft.com/en-us/pricing/details/key-vault/) and [Defender pricing](https://azure.microsoft.com/en-us/pricing/details/defender-for-cloud/) for your region and agreement. Record a Cost Management baseline. Use one dummy secret and a handful of calls; no polling loops, real AI API key, diagnostic export or extra paid protection. No keys/certificates are needed for this episode. Private Endpoint has hourly/data-processing charges, and private DNS has separate pricing; quota availability does not make these free. See [Private Link pricing](https://azure.microsoft.com/en-us/pricing/details/private-link/).

Admin PowerShell setup; run each block separately and stop if a command fails:

```powershell
$azLab = 'C:\Program Files\Microsoft SDKs\Azure\CLI2\wbin\az.cmd'
& $azLab account show --query '{name:name,id:id,tenantId:tenantId}' -o table
$subLab = & $azLab account show --query id -o tsv
$tenantLab = & $azLab account show --query tenantId -o tsv
$rgLab = 'rg-sc500-lab1'
& $azLab group show --name $rgLab --query '{name:name,location:location}' -o table
$miLab = & $azLab identity show --resource-group $rgLab --name sc500-mi --query principalId -o tsv
$adminLab = & $azLab ad signed-in-user show --query id -o tsv
```

Confirm Subscription 1 and the expected tenant. Verify the group/test user in Entra. Check their effective inherited roles; no Key Vault secret-reading role should already exist at subscription/RG scope, or the negative test will be misleading. Existing CA policies still apply to test-user sign-in; don't weaken them to get past a failure.

## 2. Create an RBAC vault with restricted public access (10 minutes)

Use Standard tier and East US to match Lab 1. Select a globally unique name of 3–24 characters, letters/digits/hyphens, beginning with a letter; no consecutive hyphens. The generated example below meets the syntax but availability must still be checked.

Determine your actual public outbound IPv4 off camera using your network/router or a trusted IP-check service. Enter it as a single-host CIDR, for example `203.0.113.10/32` illustrates the format only and is not a usable lab address. VPN/proxy changes can change egress. Key Vault IPv4 rules cannot use your local `192.168.x.x` address.

```powershell
$kvLab = 'kv-sc500-' + ([guid]::NewGuid().ToString('N').Substring(0,10))
$ipLab = Read-Host 'Enter your actual public IPv4 followed by /32'
& $azLab keyvault check-name --name $kvLab -o table
# Continue only when nameAvailable is true and the cost checkpoint is resolved.
& $azLab keyvault create --name $kvLab --resource-group $rgLab --location eastus --sku standard --enable-rbac-authorization true --retention-days 7 --enable-purge-protection true --public-network-access Enabled --default-action Deny --bypass None --network-acls-ips $ipLab --tags Lab=SC500-Lab2 Purpose=Training --query '{name:name,id:id}' -o table
$vaultScopeLab = & $azLab keyvault show --name $kvLab --query id -o tsv
& $azLab keyvault show --name $kvLab --query 'properties.{rbac:enableRbacAuthorization,publicAccess:publicNetworkAccess,firewall:networkAcls,purgeProtection:enablePurgeProtection,retention:softDeleteRetentionInDays}' -o json
```

Expected: RBAC enabled, public access Enabled, firewall default Deny, one allowed IP, bypass None, purge protection enabled, seven-day retention. In the portal, this corresponds to **selected networks**, not public access Disabled. HTTPS requests still use a public endpoint. The firewall is an additional gate; the IP rule alone grants no secret permissions.

Soft delete is enabled for new vaults. Purge protection cannot be turned off after enabling it, and deletion will retain the name/vault for the retention period. This deliberate recovery-protection lesson makes cleanup non-immediate. Use unique names for repeat recordings. See [soft delete and purge protection](https://learn.microsoft.com/en-us/azure/key-vault/general/soft-delete-overview).

## 3. Give the admin secret-management access (5 minutes)

Subscription Owner permits role assignment, but is not itself secret-read permission under RBAC. Grant the admin **Key Vault Secrets Officer** at this vault, rather than the whole RG/subscription:

```powershell
$adminRoleLab = & $azLab role assignment create --assignee-object-id $adminLab --assignee-principal-type User --role 'Key Vault Secrets Officer' --scope $vaultScopeLab --query id -o tsv
```

Allow time for authorization propagation. Review IAM > Check access. Do not add Key Vault Administrator just because the first request is delayed.

## 4. Store and retrieve a dummy secret (10 minutes)

The following string is deliberately fake and safe for the demo. Never replace it with a production AI key in a recorded terminal.

```powershell
& $azLab keyvault secret set --vault-name $kvLab --name ai-api-secret --value 'SC500-DEMO-NOT-A-REAL-API-KEY' --query '{id:id,enabled:attributes.enabled}' -o json
& $azLab keyvault secret show --vault-name $kvLab --name ai-api-secret --query value -o tsv
```

Expected: fake value returned under the **admin user**. Save metadata-only evidence if you later adapt this to actual secrets. The Azure CLI default secret output can include values, so use explicit queries and keep debug logging off.

## 5. Demonstrate control plane versus data plane (15 minutes)

| Role alone | Capability relevant to this demo |
|---|---|
| Reader | Read Azure resource configuration; cannot read secret value |
| Key Vault Contributor | Manage vault resource; does not directly grant secret-value access |
| Key Vault Reader | Inspect object metadata; cannot retrieve secret value |
| Key Vault Secrets User | Retrieve secret value; cannot create/update it |
| Key Vault Secrets Officer | Manage secret lifecycle; does not assign RBAC roles |

These are separate permissions, and effective access is cumulative across scopes/groups. The lab stays in the RBAC permission model. Switching an existing vault's permission model can break access; do not alter `myvault3`. See Microsoft's [RBAC roles and access-plane guide](https://learn.microsoft.com/en-us/azure/key-vault/general/rbac-guide).

1. With `sc500-user1` signed in to a private browser, open the vault resource. The Lab 1 group Reader allows resource visibility. Attempt to read `ai-api-secret`: value retrieval must fail. Record this before adding any data role.
2. For a CLI test, use a **separate PowerShell window with a separate CLI configuration directory**, so the admin session remains intact:

```powershell
# TEST-USER WINDOW ONLY
$env:AZURE_CONFIG_DIR = Join-Path $env:TEMP 'sc500-lab2-testuser-cli'
$azLab = 'C:\Program Files\Microsoft SDKs\Azure\CLI2\wbin\az.cmd'
& $azLab login --tenant '<tenant ID copied privately from admin window>'
# Explicitly choose sc500-user1 in the browser and complete existing MFA requirements.
& $azLab ad signed-in-user show --query userPrincipalName -o tsv
& $azLab account set --subscription '<Subscription 1 ID>'
$kvLab = '<actual vault name from admin window>'
& $azLab keyvault secret show --vault-name $kvLab --name ai-api-secret --query value -o tsv
```

3. Expected initial result: RBAC permission denial. An IP/firewall denial proves a network issue, not lack of a data role; correct the test environment before drawing a conclusion.
4. Back in the **admin window**, get the test user's object ID from Entra and grant a temporary vault-only data role directly to that test user. Keep the security group on Reader; don't grant secret-management permissions to the whole team.

```powershell
$testUserLab = Read-Host 'Paste sc500-user1 object ID from Entra'
$testRoleLab = & $azLab role assignment create --assignee-object-id $testUserLab --assignee-principal-type User --role 'Key Vault Secrets User' --scope $vaultScopeLab --query id -o tsv
```

5. After propagation, retry the read in the test-user window: the fake value should be returned. Then try this harmless write using the SAME fake value:

```powershell
# TEST-USER WINDOW ONLY: expected authorization failure; no secret version should be created.
& $azLab keyvault secret set --vault-name $kvLab --name ai-api-secret --value 'SC500-DEMO-NOT-A-REAL-API-KEY' --query id -o tsv
```

If write succeeds, inspect inherited/group roles and stop; the account has more than the intended permissions. Remove only the unintended lab grant after identifying it. Optionally insert a Key Vault Reader stage before Secrets User to demonstrate metadata-only access, removing that extra assignment afterward.

## 6. Authorize sc500-mi for future workload use (5 minutes)

In the admin window:

```powershell
$miRoleLab = & $azLab role assignment create --assignee-object-id $miLab --assignee-principal-type ServicePrincipal --role 'Key Vault Secrets User' --scope $vaultScopeLab --query id -o tsv
& $azLab role assignment list --scope $vaultScopeLab --include-inherited --query '[].{role:roleDefinitionName,principal:principalId,type:principalType,scope:scope}' -o table
```

Verify the identity's principal ID matches Lab 1. Its existing RG Reader is separate from this new secret data role. Do not run `az login --identity` on your Windows PC: there is no Azure workload identity endpoint there.

Future runtime acceptance criteria: attach this identity to a chosen Azure workload; let that workload request a token for Key Vault; read only the dummy secret and emit a success flag without the value; confirm write is denied. The workload also needs a network path allowed by the vault. Its egress won't automatically match the admin PC's allowed IP. Runtime compute/pricing and any private connectivity must be planned before deployment.

## 7. Prove network filtering independently of RBAC (10 minutes)

Leave the admin's Secrets Officer role in place. First confirm the admin can read. Then remove the PC's IP allowance:

```powershell
& $azLab keyvault network-rule remove --name $kvLab --resource-group $rgLab --ip-address $ipLab
& $azLab keyvault secret show --vault-name $kvLab --name ai-api-secret --query value -o tsv
```

After rule propagation, expect a network denial such as `ForbiddenByFirewall`. The role has not changed. The control-plane `az keyvault show` should still work because this firewall filters data-plane access.

Restore the same narrow rule immediately and confirm read works again after propagation:

```powershell
& $azLab keyvault network-rule add --name $kvLab --resource-group $rgLab --ip-address $ipLab
& $azLab keyvault secret show --vault-name $kvLab --name ai-api-secret --query value -o tsv
```

If the session is interrupted, use the control-plane network-rule add command from the admin window to restore access. Don't enable all public networks as a troubleshooting shortcut. Network settings: [Microsoft's firewall guidance](https://learn.microsoft.com/en-us/azure/key-vault/general/network-security).

## 8. Optional Private Endpoint design — not deployed in this guide

```text
Supported workload in VNet -> private endpoint NIC -> Key Vault data plane
                  DNS: <vault>.vault.azure.net resolves to endpoint private IP
                  Zone: privatelink.vaultcore.azure.net, linked to client VNet
```

A deployable private design needs a VNet/subnet, approved vault endpoint (group ID `vault`), private DNS zone and zone group, VNet DNS link, and a client inside that network or connected by an appropriate route/VPN/DNS setup. A normal internet-connected PC or Cloud Shell session is not automatically inside the VNet. Validate name resolution and a secret read from the private client before disabling public network access. A DNS record or endpoint screenshot alone does not prove private connectivity. See [Key Vault Private Link](https://learn.microsoft.com/en-us/azure/key-vault/general/private-link-service).

Quota is only an availability constraint; endpoint, DNS and client compute costs are separate. Keep this section conceptual for the initial near-zero-cost recording. A later priced extension can prove managed-identity retrieval over Private Link.

## 9. Recording plan and evidence

Suggested title: **SC-500 Lab 2: Key Vault RBAC, Managed Identity and Firewall — Who Can Read a Secret?**

| Video segment | Evidence |
|---|---|
| 0–2 min: architecture and cost | Identify the two access planes; disclose existing Defender dependency |
| 2–5 min: vault setup | RBAC, Standard, selected IP, recovery protection |
| 5–9 min: admin operation | Officer assignment, dummy secret creation/read |
| 9–14 min: user negative/positive tests | Reader denied, Secrets User reads, Secrets User write denied |
| 14–17 min: managed identity | Correct principal assignment; label runtime retrieval as pending |
| 17–20 min: firewall proof | Same admin identity denied with rule removed, allowed after restore |
| Final minutes | Explain optional private path, cleanup and cost review |

Hide object IDs/UPNs, actual egress IP, billing/account screens and authentication prompts. Use only fake values on screen. Explain that applications should prefer Entra token authentication directly to AI services when supported; Key Vault is useful when a secret is genuinely required. Storing an API key securely does not remove the need to rotate it.

## 10. Cleanup or handoff

Record the vault name, scope and exact new assignment IDs privately. PowerShell variables do not survive closing the session; recover assignment IDs from vault IAM if needed.

- Continuing the series: keep the shared Lab 1 identity/group/RG. Keep the new vault only if its ongoing cost is understood. Remove the temporary test-user Secrets User assignment using its exact ID (`az role assignment delete --ids <assignment ID>`); verify after propagation that the user again cannot read. Preserve the identity assignment only while it is needed for the later workload.
- Finished with Lab 2: remove the three lab-created vault assignments (admin Officer, test-user Secrets User, managed identity Secrets User), then delete only this lab vault in the portal. Inspect the name/RG before deletion. Do not delete the existing `myvault3` or the shared RG just to clean up Lab 2.
- Purge protection intentionally prevents immediate purge for seven days. Check Deleted vaults; do not attempt to bypass retention. Recovery does not restore all associated RBAC assignments, so recheck access after any recovery.
- In the test-user CLI window, run `az logout` via the configured CLI executable and close the window. Treat its isolated CLI config folder as credential material; do not commit it or screenshots containing tokens.
- Verify vault deletion (or documented retained state), removed temporary access, and Cost Management once usage has posted. Resource cleanup does not cancel Microsoft 365 subscriptions or change Defender subscription plans.

## Troubleshooting

| Symptom | Inspect first |
|---|---|
| Secret operation forbidden | Caller identity, effective data roles, RBAC propagation, firewall error details |
| Resource visible, secrets unavailable | Expected distinction between control plane and data plane |
| Works as admin, fails as test user | Confirm user login, CA and vault-scoped data grant |
| Both users fail after firewall change | Current public IPv4/VPN/proxy and rule propagation |
| Managed identity login fails locally | A supported attached Azure workload is required |
| Vault name unavailable after deletion | Soft-delete retention; use a new unique name |
