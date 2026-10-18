# Lab 1 — Identity, access and governance

**Duration:** 60–90 minutes, plus optional PIM. **Target incremental Azure cost:** USD 0. **Status:** Guide prepared; deployment steps have not been executed.

Outcome: demonstrate group-based Azure access, distinguish Entra roles from Azure roles, evaluate an MFA policy without enforcing it, and configure a managed identity. PIM is optional because the reviewed tenant currently lacks a suitable human-user entitlement.

## 1. Prerequisites and baseline (10 minutes)

Use Subscription 1 and the associated tenant. Your current account has Owner and Global Administrator access. These existing roles suffice; do not grant additional admin roles for the exercise. In real operations, delegate narrower roles.

Before recording:

1. In Azure Cost Management, select Subscription 1, check month-to-date actual and forecast costs, grouped by service and resource. Save a baseline. Review existing Defender trial status and Sentinel ingestion. If desired, configure alerts below USD 4 in the billing currency; alerts cannot guarantee a spending ceiling.
2. In Microsoft 365 admin center, verify Business Premium availability and renewal/expiry. Reserve one available seat for the test user. An existing entitlement avoids a new purchase, but Microsoft 365 licensing is not inherently free.
3. In Entra, inspect Security Defaults and both existing CA policies, including groups, exclusions, resources and grant controls. Do not disable existing protections for this exercise. If Security Defaults is enabled and blocks the proposed CA workflow, stop the CA portion and document the dependency instead of changing tenant-wide protection.
4. Keep the admin session open. Use a separate private browser for the test user. Have Microsoft Authenticator ready; register through the normal security-information flow if the existing policies permit it.
5. Mask UPNs, tenant/subscription IDs, billing details, MFA QR codes and credentials in the video. Record the test user creation/password screen off camera.

PowerShell session setup (the CLI was found here because `az` was absent from this process PATH):

```powershell
$azLab = 'C:\Program Files\Microsoft SDKs\Azure\CLI2\wbin\az.cmd'
& $azLab account show --query '{name:name,id:id,tenantId:tenantId}' -o table
$subLab = & $azLab account show --query id -o tsv
$rgLab = 'rg-sc500-lab1'
$scopeLab = "/subscriptions/$subLab/resourceGroups/$rgLab"
```

Confirm the displayed subscription is Subscription 1 before any writes. If a lab object name already exists, inspect it and choose a unique suffix rather than overwriting it.

## 2. Create a dedicated resource group (5 minutes)

Run when ready to deploy:

```powershell
& $azLab group create --name $rgLab --location eastus --tags Lab=SC500-Lab1 Purpose=Training
```

Expected: provisioning succeeds. Explain that this is the Azure RBAC scope for the lab. It is separate from the existing `sc500` resource group, so cleanup will not remove your existing vault/storage.

## 3. Create a test user and security group (10 minutes)

In Entra admin center:

1. Identity > Users > New user > Create new user.
2. Username `sc500-user1`, using the tenant's verified default domain; display name `SC500 Test User`. Generate a temporary password and store it privately. Leave admin roles unassigned.
3. Assign one available Business Premium license, with the Entra P1 service enabled, after setting the user's correct usage location. Verify successful provisioning.
4. Groups > New group: type **Security**, name `SC500-Security-Team`, membership **Assigned**. Add only the test user. Do not make it a role-assignable group; Azure RG Reader does not require that option.
5. Record the group object ID in your private notes.

Expected: membership shows the test user. Explain that a group is an access-management container, not an administrative role.

## 4. Assign and prove Azure RBAC (10 minutes)

Azure portal > Resource groups > rg-sc500-lab1 > Access control (IAM) > Add role assignment:

- Role: **Reader**.
- Assign access to: User, group, or service principal.
- Member: **SC500-Security-Team**.
- Scope: this resource group only.

Verification:

```powershell
& $azLab role assignment list --scope $scopeLab --include-inherited --query '[].{role:roleDefinitionName,type:principalType,principal:principalId,scope:scope}' -o table
```

Sign in as the test user in the private browser, completing any existing MFA requirements. After propagation, select the correct directory/subscription and open the RG. The empty resource group can still be inspected. Try opening a resource creation flow: Reader must not be able to deploy a resource. Capture the disabled action or authorization failure; do not grant Contributor to bypass it. If visibility fails, check membership, scope, directory selection and existing CA before changing permissions.

Success: the user can read the lab RG but cannot write. Explain: subscription Owner applies to Azure resources; Global Administrator applies to directory administration and does not automatically grant Azure resource access.

## 5. Evaluate Conditional Access in report-only (15 minutes)

Proceed only after the baseline policy/security-defaults review and P1 assignment.

Entra > Protection > Conditional Access > Policies > New policy:

- Name: `SC500-Lab1-Require-MFA-Azure-Management`.
- Users: include **SC500-Security-Team** only. Exclude the current administrator and the tenant's existing emergency-access accounts explicitly as applicable; do not create new privileged accounts for this lab.
- Target resources: select **Microsoft Azure Management** (or the corresponding current Azure management resource label in your portal). Confirm it is the Azure management resource, not all resources.
- Grant: grant access and require multifactor authentication.
- Enable policy: **Report-only**. Verify this before saving.

Use CA What If for the test user and Azure management resource. Then perform a fresh test-user Azure portal sign-in. Open Entra sign-in logs, find the relevant Azure management event and inspect **Report-only** results after logs appear. Portal sign-in can produce several resource events; choose the event that matches this policy's resource.

Expected: the policy is evaluated. It may report success if MFA is already satisfied, or another report-only result depending on the session. Report-only does not enforce MFA; an actual prompt may come from another enabled policy or Microsoft's platform requirements. What If predicts applicability but is not proof of live enforcement. Save the policy configuration and matching sign-in evidence for the video. Do not enable enforcement in this lab.

## 6. Configure a user-assigned managed identity (10 minutes)

```powershell
& $azLab identity create --name sc500-mi --resource-group $rgLab --location eastus
$miLab = & $azLab identity show --name sc500-mi --resource-group $rgLab --query principalId -o tsv
& $azLab role assignment create --assignee-object-id $miLab --assignee-principal-type ServicePrincipal --role Reader --scope $scopeLab
& $azLab identity show --name sc500-mi --resource-group $rgLab --query '{name:name,principalId:principalId,clientId:clientId}' -o table
& $azLab role assignment list --scope $scopeLab --query '[].{role:roleDefinitionName,principal:principalId,type:principalType}' -o table
```

Check each command succeeds before continuing. A newly created identity may need propagation before its role assignment succeeds.

Expected: the identity exists and has Reader on the lab RG. After this step, the test user should also be able to view the identity resource through group Reader access.

Narration: a managed identity lets a supported Azure workload obtain tokens without storing an application secret. This exercise proves identity creation and authorization configuration only. The identity is not attached to a running workload; do not claim a successful passwordless runtime call. Reader does not read Key Vault secrets or storage blobs. No VM is needed here.

## 7. Optional PIM — only with suitable licensing (15–20 minutes)

The current inventory provides P1 through Business Premium, not human-user PIM. Skip execution and show a labelled architecture explanation unless a P2/Governance entitlement or eligible trial is deliberately obtained. Check trial eligibility, renewal and licensing for all relevant participants before using it; no purchase is required for the core lab.

For a meaningful PIM demonstration:

1. Use a second, correctly licensed test user, `sc500-pim-user`, outside SC500-Security-Team, with no other Reader or broader inherited access to the lab RG.
2. PIM > Azure resources > select the lab RG. Add an **eligible Reader** assignment for that user, with a short assignment expiry. Keep scope at RG.
3. In the RG Reader activation settings, require MFA and justification and set activation maximum to one hour. Leave approval out of this short lab unless a separate approver is available.
4. Before activation, verify the user lacks effective Reader access. Activate from My roles > Azure resources, provide justification and satisfy MFA. Record activation and access after propagation.
5. Deactivate; confirm the assignment is no longer active and access is removed after propagation/session refresh. Azure authorization/token caching may delay the visible change.

Do not demonstrate PIM Reader with sc500-user1 while it already inherits permanent Reader through the security group. That would make before/after access misleading.

## 8. Evidence checklist and YouTube outline

| Segment | Approx. video time | Evidence / teaching point |
|---|---|---|
| Hook and architecture | 0:00–1:30 | “Who can access this RG, under what conditions, and using which identity?” |
| Licensing and budget | 1:30–3:00 | P1 available; PIM optional; incremental cost versus existing subscription spend |
| User, group and RBAC | 3:00–8:00 | Membership, scoped Reader, successful read and failed write |
| Conditional Access | 8:00–13:00 | Policy is report-only; What If and matching sign-in result |
| Managed identity | 13:00–16:00 | Identity principal and RG role; configuration versus runtime proof |
| Optional PIM | 16:00–20:00 | Clearly label licensed demonstration or conceptual walkthrough |
| Cleanup and recap | Last 2 minutes | Show removal, cost check and least-privilege lessons |

Suggested title: **SC-500 Lab 1: Entra ID, Azure RBAC and Conditional Access on a Tiny Budget**.

Suggested description: “Build a scoped identity lab with a test user, security group, Azure Reader access, report-only MFA Conditional Access and a managed identity. Explains P1/P2 licensing and optional PIM. No VM deployment. Costs and existing license availability vary.”

## 9. Cleanup

In this order, inspect each exact object before removing it:

1. Delete only `SC500-Lab1-Require-MFA-Azure-Management`; leave the two pre-existing CA policies alone.
2. Remove optional PIM eligible/active assignments and the optional PIM test user if created. Remove any separately created license assignments. A trial subscription, if started independently, needs its own renewal/cancellation review.
3. Delete rg-sc500-lab1 in Azure portal after confirming it contains only the new lab identity. This deletes the managed identity; verify its role assignment is gone and remove any orphaned assignment by its exact ID if needed.
4. Delete the lab security group and test user; reclaim the test user's Business Premium seat. These directory objects are not deleted by RG deletion.
5. Verify the lab RG and lab CA policy no longer exist; recheck Cost Management when usage has posted. Keep the existing `sc500`, Sentinel, NetworkWatcher and AI resource groups intact.

## References

- SC-500 objectives: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/sc-500
- CA and licensing: https://learn.microsoft.com/en-us/entra/identity/conditional-access/overview
- Report-only: https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-conditional-access-report-only
- PIM licensing: https://learn.microsoft.com/en-us/azure/active-directory/governance/licensing-fundamentals
- Managed identities and no-extra-cost identity feature: https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview
