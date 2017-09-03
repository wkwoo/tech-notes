Red Hat OpenStack Platform 8 Installation and Deployment
========================================================
* Use an OpenStack (aka Undercloud, for deployment) to deploy another OpenStack (aka Overcloud, for tenants). Called OpenStack on OpenStack (Triple-O).
* RH OpenStack Platform Director (Director) is the tool in the case of RHOSP.
* Undercloud node - Director, a single system OpenStack installation.
* Overcloud nodes - Controller (HA), Compute, Storage

## Environment Requirements
### Minimum Requirements
1. 1 * Director node
2. 1 * Controller node
3. 1 * Compute node

### Recommended Requirements
1. 1 * Director node
2. 3 * Controller nodes (in an HA cluster)
3. 3 * Compute nodes
4. 3 * Ceph storage nodes (in an HA cluster)

**Notes** for production deployment:
* Bare metal for all nodes if possible
* At least compute node on bare metal
* All Overcloud bare metal system need Intelligent Platform Management Interface (IPMI) for Director to control power management of these nodes.

### Undercloud Host System HW & SW Requirements
1. CPU: 8-core x86_64
2. RAM: >= 16 GiB
3. Disk: >= 40 GiB. 
  * Min. 10 GiB free before attempting an Overcloud deployment or update
  * Needed for image conversion and caching during node provisioning.
4. Network interface card: 
  * Min. 2 * 1 Gbps NIC
  * Recommended 10 Gbps for Provisioning network
5. Operating system: RHEL-7.2 or newer
6. File system (when using LVM) partition: 
  * Contains **only** a root (/) and swap partitions. 
  * [Director node fails to boot after undercloud installation](https://access.redhat.com/solutions/2327921)

### Undercloud Networking Requirements
1. Provisioning Network
  * Private network - for Director to provision and manages Overcloud nodes
  * DHCP & PXE boot (so must use a native VLAN or truncked interface) (???)
  * IPMI - to manage Overcloud nodes power
2. External Network
3. NIC configurations
  1. 1 NIC
    * Provisioning network on native VLAN
    * Tagged VLANs (use subnets) for Overcloud networks
  2. 2 NIC
    * 1 for Provisioning network
    * 1 for External network
  3. 2 NIC
    * 1 for Provisioning network on native VLAN
    * 1 for Overcloud network with tagged VLANs and subnets
  4. Multiple NICs - each NIC use different subnet for different Overcloud network
  5. Additional NIC for 
    * Isolating individual networks
    * Creating bonded interfaces
    * Delegating tagged VLAN traffic
  6. Tagged VLANs need switches that support 802.1Q standards
  7. Use consistent/fixed NIC name across all Overcloud machines to avoid confusion. I.e. primary NIC for Provisioning network and secondary NIC for OpenStack services.
  
  
