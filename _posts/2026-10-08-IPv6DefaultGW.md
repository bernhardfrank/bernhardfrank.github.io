---
title:  "Azure Local: 'Create the cluster' fails - blame it on IPv6 ;-)"
date:   2026-10-08
tags: [Azure Local, Networking]
---

## Summary

I changed my setup to a single dual port NIC due to a HW change. Thus I ran the setup with all traffic classes (mgmt, compute and storage) on these 2 ports only and ran into an error.  
It turned out that the storage vNics picked up an IPv6 Default Gateway which broke the installation.  
Keep on reading if you are interested in the full story:  

## Scenario

Fresh Azure Local deployment with **build 2609** in my lab:

- 2 nodes (HCIMX1, HCIMX2), **switched**
- **Fully converged network** - i.e. **one network intent** (`compute_management_storage`) with only **2 ports per node** (Intel X722, 10 GbE) carrying all traffic: Management, Compute and Storage
- Management untagged (native VLAN 777 on the switch), storage on the default VLANs **711 / 712**
- The switch (Lenovo G8272, CNOS) is also the default gateway for management (`interface Vlan777` - 172.31.254.254)

Nothing fancy - a pretty standard setup IMHO.

## Problem

The deployment runs smoothly through all validation and configuration steps... until **'Create the cluster'**:
![deployment fails at create the cluster](/assets/images/posts/2026/10/IPv6DefGtwy-00.png)

The details say:
![error details in the portal](/assets/images/posts/2026/10/IPv6DefGtwy-01.png)

> *Failed to create the storage cluster 'HCIMX'. New-Cluster error: Network 10.71.1.0/24 without DHCP was assigned role ClusterAndClient, but no address was given to configure the Cluster Name on this network. [...] Networks with default gateways are assigned ClusterAndClient role by default.*

Hmm... 10.71.1.0/24 is my **storage** network (the automatic IPs Network ATC assigns to the vSMB adapters). Why on earth would the cluster want to put its Cluster Name onto storage?

## Troubleshooting

### Log files

The deployment logs (`C:\CloudDeployment\Logs` on the seed node) show the same - the cluster validation ran fine (with some warnings), then `New-Cluster` was called with only the reserved management IP (`-StaticAddress 172.31.0.20`) and failed:
![deployment log](/assets/images/posts/2026/10/IPv6DefGtwy-02.png)

The interesting part of the message is the last sentence: **"Networks with default gateways are assigned ClusterAndClient role by default."**  
But **I want my storage networks only to do Cluster only stuff**.  
So somewhere the storage network must have a default gateway - which it shouldn't.

### Get-NetIPConfiguration

First guess: some IPv4 gateway on a storage adapter. So let's check on both nodes:

```powershell
Get-NetIPConfiguration
```

![Get-NetIPConfiguration shows an IPv6 default gateway on the vSMB adapters](/assets/images/posts/2026/10/IPv6DefGtwy-03.png)

No IPv4 gateway on the vSMB adapters - good. **But** look at the `IPv6DefaultGateway`: both storage adapters (vSMB#COMP1 and vSMB#COMP2) have `fe80::a68c:dbff:fe3d:c501` - **the very same router as vManagement**. Same picture on node 2.

Some more checks to rule out the obvious:

```powershell
# VLAN tagging of the host vNICs
Get-VMNetworkAdapterIsolation -ManagementOS | Select ParentAdapter, IsolationMode, DefaultIsolationID
# IPv6 default routes
Get-NetRoute -AddressFamily IPv6 -DestinationPrefix ::/0 | Select InterfaceAlias, NextHop, ValidLifetime
```

- vManagement: untagged (-> native VLAN 777), vSMB: VLAN 711 / 712 - **correct**.
- Switch: VLANs 711/712 are pure layer 2 (no SVI), PFC/ETS for priority 3 (RoCE) and 7 (cluster) - **correct**.

Side note: the link-local address even tells you who the router is - `fe80::a68c:dbff:fe3d:c501` -> MAC `A4-8C-DB-3D-C5-01` (EUI-64, flip the 7th bit) - a Lenovo MAC, i.e. my switch.

## Reason

**IPv6 Router Advertisements (RAs).**

- CNOS sends IPv6 Router Advertisements on its SVIs - also on `interface Vlan777`, my management VLAN, which is the **native VLAN** on the node ports. Nobody configured IPv6 there... it is just on by default.
- During deployment Network ATC creates the host vNICs and **tags the vSMB adapters with VLAN 711/712 shortly afterwards**. In between they are untagged - i.e. sitting in VLAN 777 - and happily pick up the switch's RA. That gives them an **IPv6 default route** (`::/0`) which stays for the router lifetime, even after the VLAN tag is applied.
- **Failover Clustering doesn't care if a default gateway is IPv4 or IPv6.** A network with a default gateway gets the role **ClusterAndClient**, and a ClusterAndClient network needs an IP for the Cluster Name.
- The deployment only reserved a cluster IP for management (172.31.0.20) -> `New-Cluster` has no address for 10.71.1.0/24 -> **fail**.

So nothing in the Azure Local configuration was wrong - an IPv6 feature nobody uses, combined with a timing issue in a fully converged setup.

## Fix

Since I don't use IPv6 in this network, I simply turned off the Router Advertisements on the management VLAN interface of the switch:

```
G8272due(config)#interface vlan 777
G8272due(config-if)#ipv6 nd suppress-ra
```

![suppress RA on interface Vlan777](/assets/images/posts/2026/10/IPv6DefGtwy-04.png)

After a while (or after removing the stale `::/0` routes) the IPv6 default gateways are gone on all adapters:
![no IPv6 default gateway anymore](/assets/images/posts/2026/10/IPv6DefGtwy-05.png)

Then simply hit **'Resume deployment'** in the portal:
![resume deployment](/assets/images/posts/2026/10/IPv6DefGtwy-06.png)

...and the cluster deployed successfully.

### If you can't touch the switch  

>**Warning: This section is experimental an was suggested by claude. I haven't tried it but found it useful in this context. So no warranties!**

Alternatively you can stop the storage vNICs from listening to RAs on the nodes and remove the learned routes (run on any node):

```powershell
Invoke-Command -ComputerName HCIMX1, HCIMX2 -ScriptBlock {
    Get-NetIPInterface -InterfaceAlias "vSMB(*" -AddressFamily IPv6 |
        Set-NetIPInterface -RouterDiscovery Disabled
    Get-NetRoute -AddressFamily IPv6 -DestinationPrefix ::/0 -ErrorAction SilentlyContinue |
        Where-Object InterfaceAlias -like "vSMB(*" |
        Remove-NetRoute -Confirm:$false
}
```

However fixing it at the source (the switch) is the cleaner way - it also prevents the issue on redeployments or when adding nodes.

### **Takeaway:** If cluster creation complains about a *ClusterAndClient* role on your storage network - don't only look for IPv4 gateways. Check `IPv6DefaultGateway` too ;-)
