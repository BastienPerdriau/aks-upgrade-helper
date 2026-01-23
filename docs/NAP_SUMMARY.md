# Summary: NAP User-Assigned Managed Identity Research

## Question
Can we attach additional user-assigned managed identities to nodes provisioned by NAP (Node Auto Provisioning) in AKS?

## Answer
**NO** - It is not currently possible.

## Key Points

1. **Current Status (January 2026)**
   - AKS NAP does NOT support attaching user-assigned managed identities to nodes
   - This is a documented limitation tracked on Azure's roadmap
   - GitHub Issue: [Azure/AKS#5496](https://github.com/Azure/AKS/issues/5496)

2. **Why It Matters**
   - Traditional AKS node pools can have custom managed identities configured on VMSS
   - NAP uses Karpenter for dynamic node provisioning
   - NAP defaults to cluster's managed identity only - no custom identity support

3. **Impact**
   - Workloads requiring specific node-level identities will fail on NAP nodes
   - Breaks scenarios where different workloads need different Azure resource access

4. **Recommended Workaround**
   - Use **Azure Workload Identity** (OIDC-based, modern approach)
   - Assigns identities at pod-level instead of node-level
   - Works seamlessly with NAP-provisioned nodes
   - More secure and granular than node-level identities

5. **Alternative Workarounds**
   - Use traditional node pools for identity-dependent workloads
   - Use NAP only for identity-agnostic workloads
   - Hybrid approach with node selectors and taints/tolerations

## Recommendation for AKS Upgrade Helper

**Do NOT add a NAP-specific check at this time**

Reasons:
- NAP is optional (most clusters don't use it)
- Feature is in development
- Pod-level identities (recommended approach) work fine with NAP
- Not an upgrade-blocking issue

## Full Documentation

See [NAP_MANAGED_IDENTITY_FINDINGS.md](./NAP_MANAGED_IDENTITY_FINDINGS.md) for complete analysis, workarounds, and references.

## References

- [Microsoft Docs: NAP Overview](https://learn.microsoft.com/en-us/azure/aks/node-auto-provisioning)
- [GitHub: Feature Request](https://github.com/Azure/AKS/issues/5496)
- [Microsoft Docs: Azure Workload Identity](https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview)
