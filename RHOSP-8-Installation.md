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

## Deploying Undercloud (Director)
Create a director installation user 'stack'
 * Non-root
 * password-less sudo access

**Perform the following as the stack user.**
1. Create directories in stack's home directory
    * `~/images`
    * `~/templates`
2. Set hostname for the director (use `hostnamectl`)
3. Add director host name (both FQDN and alias) into `/etc/hosts` (to 127.0.0.1)
    * Must set or the script will fail, even if a DNS is available
4. Register to RH CDN or prepare the director with the following Yum repositories
    * base (from RHEL-7 installation DVD)
    * rhel-7-server-rpms (from OSP-8 installation DVD)
    * rhel-7-server-extras-rpms (from OSP-8 installation DVD)
    * rhel-7-server-openstack-8-rpms (from OSP-8 installation DVD)
    * rhel-7-server-openstack-8-director-rpms (from OSP-8 installation DVD)
    * rhel-7-server-rh-common-rpms (from OSP-8 installation DVD)
5. `sudo yum update -y; sudo reboot`
6. `sudo yum install -y python-tripleoclient`
7. Prepare a local CA in director host (if using self-signed certs)
    * See [Preparing a Local CA](https://github.com/wkwoo/tech-notes/blob/master/RHOSP-8-Installation.md#preparing-a-local-ca)
8. `cp /usr/share/instack-undercloud/undercloud.conf.sample ~/undercloud.conf`
9. Upate the `~/ca/undercloud.conf` file with following parameters
    * `local_ip`
    * `network_gateway`
    * `undercloud_public_vip`
    * `undercloud_admin_vip`
    * `undercloud_service_certificate` (needs cert from step #7)
    * `local_interface`
    * `network_cidr`
    * `masquerade_network`
    * `dhcp_start`, `dhcp_end`
    * `inspection_interface`
    * `inspection_iprange`
    * `inspection_extras`
    * `inspection_runbench`
    * `undercloud_debug`
    * `enable_tempest`
    * `ipxe_deploy`
    * `store_events`
    * Passwords: 
        * `undercloud_db_password`, 
        * `undercloud_admin_token`, 
        * `undercloud_admin_password`, 
        * `undercloud_glance_password`, 
        * etc
10. `openstack undercloud install` - The installation will take quite a while.

### To Verify the Undercloud Installation
**Perform the following as the stack user.**
1. Source the stackrc profile: `. ~/stackrc`
2. `openstack endpoint list`
3. `openstack host list`
4. `python -m json.tool /etc/os-net-config/config.json`
5. `sudo ovs-vsctl show`
6. `cat /etc/sysconfig/network-scripts/ifcfg-br-ctlplane`
7. `ip a`

## Configuring Undercloud After Installation
**Perform the following as the stack user.**

### Preparing Images for Overcloud Nodes
1. `sudo yum install rhosp-director-images rhosp-director-images-ipa`
2. `cp /usr/share/rhosp-director-images/overcloud-full-latest-8.0.tar ~/images/.`
3. `cp /usr/share/rhosp-director-images/ironic-python-agent-latest-8.0.tar ~/images/.`
4. `cd ~/images`
5. `for tarfile in *.tar; do tar -xf $tarfile; done`
6. `openstack overcloud image upload --image-path /home/stack/images/`
7. `openstack image list`
8. `ls -l /httpboot`

### Setting a Nameserver on the Undercloud's Neutron Subnet
1. `neutron subnet-list`
2. `neutron subnet-update <subnet-uuid> --dns-nameserver <nameserver-ip>`. For example, assuming 192.168.101.2 has been configured as a DNS server: `neutron subnet-update 42178b17-68ea-452f-9ee2-b02ea87a27e9 --dns-nameserver 192.168.101.2`
3. `neutron subnet-show <subnet-uuid>`

## Preparing a Local CA
Perform the following **as the `stack` user**, in the `ca` directory (create one if not exists)
1. `openssl genrsa -out ca.key.pem 4096`
2. `openssl req -key ca.key.pem -new -x509 -days 7300 -extensions v3_ca -out ca.crt.pem`
3. `sudo cp ca.crt.pem /etc/pki/ca-trust/source/anchors/`
4. `sudo update-ca-trust extract`
5. `sudo cp ca.key.pem /etc/pki/CA/private/`
6. If does not exists: `sudo touch /etc/pki/CA/index.txt`
7. If does not exists: `sudo sh -c 'echo 1000 > /etc/pki/CA/serial'`
8. `cp /etc/pki/tls/openssl.cnf ~/ca/`
9. Update the `~/ca/openssl.cnf` for following:
    * `[ CA_default ]`
       - `private_key = $dir/private/ca.key.pem`
    * `[ req ]` 
       - `distinguished_name = req_distinguished_name`
       - `req_extensions = v3_req`
    * `[ req_distinguished_name ]`
        - Update:
            - `countryName_default`
            - `stateOrProvinceName_default`
            - `localityName_default`
            - `organizationalUnitName_default` (if applicable)
        - Make sure that:
            - `commonName_default = <same as undercloud_public_vip>` (for undercloud)
            - `commonName_default = <1st address of ExternalAllocationPools>` (for overcloud) 
    * `[ v3_req ]`
        - `subjectAltName = @alt_names`
    * `[ alt_names ]` - `IP.1` and `DNS.1` **must** have the same IP address 
        - `IP.1 = <same IP used in req_distinguished_name.commonName_default>`
        - `DNS.1 = <same IP used in req_distinguished_name.commonName_default>`
        - `DNS.2 = (optional) alternative name of the host`
        - `DNS.3 = (optional) alternative name of the host`
10. `openssl genrsa -out server.key.pem 2048`
11. `openssl req -config openssl.cnf -key server.key.pem -new -out server.csr.pem`
12. `sudo openssl ca -config openssl.cnf -extensions v3_req -days 3650 -in server.csr.pem -out server.crt.pem -cert ca.crt.pem`

**For Undercloud**
1. `cat server.crt.pem server.key.pem > undercloud.pem`
2. `sudo mkdir /etc/pki/instack-certs`
3. `sudo cp undercloud.pem /etc/pki/instack-certs/`
4. `sudo semanage fcontext -a -t etc_t "/etc/pki/instack-certs(/.\*)?"`
5. `sudo restorecon -Rv /etc/pki/instack-certs/`
6. The resulting undercloud.pem is to be used as value to undercloud_service_certificate in undercloud.conf, i.e.
`undercloud_service_certificate = /etc/pki/instack-certs/undercloud.pem`

**For Overcloud**
TODO
 
