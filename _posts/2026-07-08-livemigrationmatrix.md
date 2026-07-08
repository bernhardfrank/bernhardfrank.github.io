---
title:  "Enter the LiveMigration Matrix"
date:   2026-07-08 
tags: [Hyper-V]
---

## Using Hyper-V - between which versions can I do a Live Migration?

I've got this question by a friend and it somehow stayed in my head. Simple question, isn't it?  
The more I thought the more difficult it appeared to me. (Maybe I should have thought a bit more thoroughly ;-))  
My thinking was that: Although the settings in the tools e.g. Hyper-V Manager have not changed much. (i.e. Authentication protocol (CredSSP, Kerberos), performance option (TCP/IP, Compression, SMB)) there might be other changes that would prevent VMs to be live migrated.
  
So to get it out of my head again...and as some kind of practical guy I created the following test setup:  
- On 2x Dell R730 - I installed Windows Server 2025 with Failover cluster and Hyper-V (S2D) - I then created a bunch of nested (i.e. virtualized) Hyper-V VMs inside running Windows Server 2016 ... Windows Server 2025 with Hyper-V enabled.
![the livemigration test setup](/assets/images/posts/2026/hyperv-failover-cluster.svg)
- Thus all nested Hyper-V VMs (HV20xx-y) share the same underlying physical HW - so no incompatibilities e.g. CPU from that side. 
- I created 2 Hyper-V server VMs of each OS version (e.g. HV2016-1, HV2016-2,...,HV2025-1, HV2025-2) in total 8 servers with 4 on each physical node:
![nested Hyper-V VMs](/assets/images/posts/2026/R730-225.png)
- All Hypervisors 'HV20xx-y' are domain joined. 4 vNics(MGMT, Compute1 Compute2, SMB1, SMB2), standalone (not clustered), LM enabled using Kerberos delegatgion + SMB:
![Hyper-V LM Settings](/assets/images/posts/2026/HV2016LMSettings.png)
- LM traffic flow enabled on MGMT, SMB1 and SMB2 networks. 
- As workload VM to be moved around: I have created a small alpine Linux VM as Gen2 VM created on every Hyper-V OS Version - i.e. 4 VMs: alpineCV8, alpineCV9, alpineCV10 and alpineCV12. Connected to my MGMT network thus I could ping it anywhere in my domain.

Then things got funny - doing live migrations like mad...  
 ![live migrating ](/assets/images/posts/2026/lmigrating.png)  
 ('shared nothing live migration' - i.e. copy virtual harddisk of the vms to destination)  
   
...well, not really...  
**...Live Migrations just work fine between different Hyper-V versions - if the destination Hyper-V system understands the to-be-migrated-VM's 'Configuration Version'.**  
![ Hyper-V 2025 runs different configuration versions ](/assets/images/posts/2026/confversion.png)  
Somehow very logical and I could have thought about it upfront - saving me time - it was somewhere in my brain.  
However as you might stumble upon it as well...  

## Here it is the final tested Live Migration Matrix for Hyper-V 20xx to Hyper-V 20yy: 

| dest-v   /   source-> | from Hyper-V 2016 | from Hyper-V 2019 | from Hyper-V 2022 | from Hyper-V 2025|
| --- | --- | --- | --- | --- |
| to Hyper-V 2016 | CV8 | CV8 | CV8 | CV8 |
| to Hyper-V 2019 | CV8 | CV8/CV9 | CV8/CV9 | CV8/CV9 |
| to Hyper-V 2022 | CV8 | CV8/CV9 | CV8/CV9/CV10 | CV8/CV9/CV10 |
| to Hyper-V 2025 | CV8 | CV8/CV9 | CV8/CV9/CV10 | CV8/CV9/CV10/CV12 |

- CV8: VMs created on Hyper-V 2016
- CV9: VMs created on Hyper-V 2019
- CV10: VMs created on Hyper-V 2022
- CV12: VMs created on Hyper-V 2025

As Hyper-V developed over the time so did the 'Configuration Version' (increment) - as it makes the latest Hyper-V features available on your virtual machine. (e.g. 'Hibernation support' CV9, 'GPU partitioning' CV12)  
Imported (also live migrated) VMs can be upgraded to the latest configuration version (when off) available on it's executing Hyper-V system.  
However **before you upgrade the configuration version of a Hyper-V VM make sure**...  
...**that you won't need to move the virtual machine back to a Hyper-V host that runs a previous version** of Windows or Windows Server.
[Upgrade virtual machine version in Hyper-V on Windows or Windows Server](https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v/deploy/upgrade-virtual-machine-version-in-hyper-v-on-windows-or-windows-server)

