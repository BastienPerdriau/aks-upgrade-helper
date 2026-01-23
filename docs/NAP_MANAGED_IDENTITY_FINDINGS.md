# NAP (Node Auto Provisioning) and User-Assigned Managed Identity Support

## Executive Summary

**Current Status:** Azure Kubernetes Service (AKS) Node Auto-Provisioning (NAP) **does NOT currently support** attaching additional user-assigned managed identities to nodes that it provisions.

## What is NAP?

Node Auto-Provisioning (NAP) is an AKS feature that dynamically provisions nodes using Karpenter. It optimizes for cost and performance by matching resource requirements of pending pods to the best possible VM configuration.

## The Problem

### Current Limitation

As of January 2026, AKS Node Auto-Provisioning does not natively support attaching user-assigned managed identities to newly provisioned nodes. This is a tracked limitation in the Azure AKS product roadmap.

**GitHub Issue:** [Azure/AKS#5496 - Feature: Add support for custom managed Identities with NAP](https://github.com/Azure/AKS/issues/5496)

### Why This Matters

In traditional AKS node pools:
- Administrators can configure user-assigned managed identities on Virtual Machine Scale Sets (VMSS)
- Workloads running on those nodes can use the identity to access Azure resources securely
- Different node pools can have different managed identities for different workload requirements

With NAP:
- Karpenter provisions nodes on-demand dynamically
- Currently defaults to using the cluster's managed identity only
- **No mechanism exists to dynamically assign different user-managed identities per node**
- Workloads that depend on specific node-level managed identities will fail

### Impact

If you deploy workloads that require a specific user-assigned managed identity and Karpenter provisions the node:
- The custom identity will NOT be attached to the node
- Pods will not find the expected identity
- Resource access that depends on that identity will be broken

## Current Configuration Capabilities

The AKS Karpenter documentation for `AKSNodeClass` allows configuration of:
- ✅ VM size and SKU
- ✅ Subnet assignment
- ✅ OS image selection
- ✅ OS disk configuration
- ❌ **Custom user-assigned managed identities** (NOT supported)

## Workarounds

Until Microsoft adds native support for user-assigned managed identities with NAP, consider these alternatives:

### 1. Use Traditional Node Pools for Identity-Dependent Workloads
- Keep workloads that require specific user-assigned identities on traditional VMSS-based node pools
- Manually configure the required identity on those node pools
- Use NAP for general-purpose, identity-agnostic workloads

### 2. Adopt Pod-Level Identity Management (Recommended)
Instead of assigning identities at the node level, assign them directly to pods:

**Azure Workload Identity (Recommended)**
- Modern, OIDC-based approach
- Assigns identities directly to specific pods
- Works seamlessly with NAP-provisioned nodes
- More granular security model

**aad-pod-identity (Legacy)**
- Older approach, being phased out
- Still functional but Azure Workload Identity is preferred

### 3. Hybrid Approach
- Use NAP for general workloads
- Use traditional node pools with custom identities for specific workloads
- Leverage node selectors and taints/tolerations to control pod placement

## Recommendations for AKS Upgrade Helper

### Should We Add a Check?

Given the current state of NAP support in AKS:

**Recommendation: NO - Don't add a NAP-specific check at this time**

**Reasons:**
1. **NAP is an optional feature** - Most AKS clusters don't use NAP
2. **Feature is in preview/development** - Microsoft is actively working on this
3. **Validation complexity** - Would require detecting NAP usage, then checking for user-assigned identities, then determining if those identities are needed at the node level vs. pod level
4. **Workarounds exist** - Pod-level identities are the recommended approach anyway
5. **Limited impact on upgrades** - This is a feature limitation, not an upgrade-blocking issue

### If a Check is Needed in the Future

If NAP adoption increases and this becomes a common upgrade concern, consider adding a check that:

1. Detects if the cluster has NAP enabled
2. Checks if NAP is using user-assigned managed identities (when supported)
3. Validates identity configuration before upgrade
4. Warns users about potential identity-related issues

Example check structure:
```bash
function run_check() {
    log_info "Checking NAP managed identity configuration..."
    
    # Check if NAP is enabled
    nap_enabled=$(echo $CLUSTER_JSON | jq -r '.properties.nodeResourceGroupProfile.restrictScaleSetToVnet // false')
    
    if [[ "$nap_enabled" != "true" ]]; then
        log_info "NAP is not enabled on this cluster"
        return 0
    fi
    
    # Future: Check for user-assigned identities when supported
    # ...
    
    return 2  # Warning until feature is GA
}
```

## Staying Updated

To monitor the status of this feature:

1. **GitHub Issue Tracker**
   - Watch: https://github.com/Azure/AKS/issues/5496
   - Check for updates and announcements

2. **Azure Documentation**
   - [Overview of node auto-provisioning (NAP)](https://learn.microsoft.com/en-us/azure/aks/node-auto-provisioning)
   - [Configure AKSNodeClass resources](https://docs.azure.cn/en-us/aks/node-auto-provisioning-aksnodeclass)

3. **AKS Release Notes**
   - Monitor monthly AKS release notes for feature announcements
   - Check preview features for early access

## References

- Microsoft Learn: [Overview of node auto-provisioning (NAP) in Azure Kubernetes Service](https://learn.microsoft.com/en-us/azure/aks/node-auto-provisioning)
- GitHub: [Feature Request - Add support for custom managed Identities with NAP](https://github.com/Azure/AKS/issues/5496)
- Azure Docs: [Configure AKSNodeClass resources for NAP](https://docs.azure.cn/en-us/aks/node-auto-provisioning-aksnodeclass)
- Azure Docs: [Use Azure Workload Identity with AKS](https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview)

## Conclusion

**Answer to the question: "Is it possible to attach additional user-assigned managed identity to nodes provisioned by NAP?"**

**NO** - It is not currently possible to attach additional user-assigned managed identities to nodes provisioned by NAP. This is a documented limitation that Microsoft is aware of and tracking on their roadmap.

**Recommended Alternative:** Use Azure Workload Identity to assign managed identities at the pod level instead of the node level. This is the modern, recommended approach and works seamlessly with NAP-provisioned nodes.

**For AKS Upgrade Helper:** No immediate action is needed. This is a feature limitation rather than an upgrade-blocking issue. Monitor the GitHub issue and Azure documentation for updates.

---

*Document Created: January 23, 2026*
*Last Updated: January 23, 2026*
