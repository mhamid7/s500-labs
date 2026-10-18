# SC-500 tenant review and five-lab plan

Reviewed 18 October 2026 using read-only Azure CLI and Microsoft Graph queries. No Azure or Entra configuration was changed. The attached diagram was treated as a proposed design, not permission to execute its steps.

## Readiness

- One accessible subscription: **Subscription 1**, Enabled and the CLI default.
- Signed-in account has subscription **Owner**, root-scope User Access Administrator, and active Entra **Global Administrator** and Billing Administrator membership. Azure and Entra roles are separate permission systems.
- Microsoft 365 Business Premium (`SPB`): Enabled, 25 seats, 1 consumed; its `AAD_PREMIUM` service plan is provisioned. Assign a suitable available license to the lab user before Conditional Access testing. Check subscription expiry/renewal in Microsoft 365 billing; the SKU inventory does not establish whether these seats are paid or trial.
- No human-user Entra P2 or Entra ID Governance entitlement was found. Agent 365 governance plans do not establish entitlement to human-user PIM. A standalone P1 SKU is suspended, but Business Premium supplies an enabled P1 entitlement.
- Existing Conditional Access: `Policy1` is report-only; `Non-Compliant-365-Block` is enabled. Their targeting and interaction need inspection before a test sign-in.
- Security Defaults status could not be read: Graph returned missing token scopes. This is an API permission limitation, not proof that the feature is disabled. Inspect it in Entra admin center before adding CA.

## Azure inventory

| Resource group | Resources | Region |
|---|---|---|
| sc500 | gazidentity (user-assigned identity), myvault3 (Key Vault), mygaz01 (storage), sc500-vnet (VNet), office365 (API connection) | East US |
| Microsoft-Sentinel-Resource-Group | Microsoft-Sentinel-Workspace and its SecurityInsights solution | East US |
| NetworkWatcherRG | NetworkWatcher_eastus | East US |
| DefaultResourceGroup-USW3 | maceddie-2875-resource (Cognitive Services account) | Account East US 2; group West US 3 |

No VM, Azure Firewall, Bastion, private endpoint, or AKS resource appeared in this inventory. Resource existence does not prove that an identity is attached to a workload or that a connector is collecting data.

## Cost findings

Month-to-date ActualCost query returned USD **0.0000506** for Storage and USD 0 for Defender for Cloud. This is reported pre-tax cost, not a final invoice or future-cost forecast. M365 subscription charges are separate.

Defender subscription pricing shows Standard for VirtualMachines/P2, AppServices, KeyVaults/PerKeyVault, CloudPosture, AI, Discovery and FoundationalCspm. Standard does not by itself prove billable usage: some capabilities are free and trial/resource coverage varies. Inspect coverage, trial expiry and cost by resource before deploying workloads. Sentinel ingestion, storage and AI usage also warrant review.

Working assumption: **less than USD 4 additional Azure spending for the whole five-lab series**, excluding existing M365 licensing. Lab 1 targets USD 0 additional usage. A budget alert is not a hard spending cap. Do not start paid licenses, Defender plans, AI deployments, or metered infrastructure just to follow a video.

## Corrections to the supplied diagram

1. Replace “$0 uses free features” with “Core Azure identity/RBAC steps: no additional charge; CA requires P1; PIM requires P2 or Governance.”
2. Report-only Conditional Access evaluates a policy; it does not require MFA during sign-in.
3. Scope PIM to the lab resource group rather than the subscription. Do not make the same user's Reader access both permanent through a group and eligible through PIM: permanent access would hide the activation effect.
4. Managed identity Reader access is control-plane metadata access. It does not grant Key Vault secret or blob data access, and a user-assigned identity needs a supported workload to obtain tokens. Lab 1 verifies configuration only.
5. RG location stores RG metadata; it does not force contained resources into that region. Use East US for this lab to match existing work, unless teaching a UAE-specific deployment.

## Proposed episodes

| Lab | Content and proof | Budget approach |
|---|---|---|
| 1 | Entra user/group, RG Reader, report-only MFA CA, managed identity; optional PIM | No new metered Azure service; use existing P1 seats |
| 2 | Key Vault + managed identity + RBAC + IP firewall: separate resource access from secret access | New isolated Standard vault only after Defender pricing review; tiny dummy-secret operations; Private Endpoint conceptual |
| 3 | Network governance: VNet/NSG, Policy audit, locks; explain private access | No VMs or paid network appliances; use audit assignments and configuration evidence |
| 4 | Compute and AI security posture | Read existing Defender recommendations and AI settings; demonstrate configuration/review, not paid workload protection deployment |
| 5 | Monitoring and investigation: Activity Log, Defender posture, existing Sentinel | Query existing data; no new ingestion, connectors, playbooks or paid Copilot capacity |

This is a budget-focused series, not complete hands-on coverage of every SC-500 objective. Paid technologies can be explained using diagrams and clearly labelled walkthroughs. Lab 4 especially must distinguish inspecting settings from proving runtime protection.

## Official references

- Exam scope: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/sc-500
- CA: https://learn.microsoft.com/en-us/entra/identity/conditional-access/overview
- PIM licensing: https://learn.microsoft.com/en-us/azure/active-directory/governance/licensing-fundamentals
- Managed identities: https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview
- Cost alerts: https://learn.microsoft.com/en-us/azure/cost-management-billing/costs/cost-mgt-alerts-monitor-usage-spending
