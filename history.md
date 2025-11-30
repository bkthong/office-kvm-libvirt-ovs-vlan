[20251121]
- Faikam isntalled 10Gbps NIC AQC-107 (atlantic kernel module is the driver
  auto-detected
- For testing eno2 was connected to a trunk port on the switch with vlan 
  21-26 (around that range)
- Use ```tcpdump -i NIC -s 0 -e -nn vlan```
    - Capture vlan tagged packets ("vlan" filter)
    - On the specified NIC (-i)
    - Don't limit packet size (-s 0)
    - Show ethernet header (-e) which contains the vlan info
    - Numeric format (-nn)

[20251122]
- Cocluded that openvswitch is the way to go rather than linux bridge
  as we can set vlan tag at the vm guest level.

[20251125]
- As linux bridges are actually trunks by default and we can make a vlan
  a port on the bridge.
- Due to this, final decision was to go for linux brodge and vlan
  for easier maintanence. Details in below sections

> **FINAL DECISION WAS LINUX Bridge and vlan WITHOUT OVS**. Discussion 
  on ovs are left here for ref and perhaps maybe future needs

# Overview
- libvirt 11 supports vlan tagging at VM xml 

> Network connections that support guest-transparent VLAN tagging include type='bridge' interfaces connected to an Open vSwitch bridge, SRIOV Virtual Functions (VF) used via type='hostdev' (direct device assignment), since 1.3.5, SRIOV VFs used via type='direct' with mode='passthrough' (macvtap "passthru" mode) and, since 11.0.0, standard linux bridges.  [network xml](https://libvirt.org/formatnetwork.html) [domain/vm xml](https://libvirt.org/formatdomain.html)

- **BUT** Rocky 10 (this host) is using libvirt 10

- Hence we need to use openvswitch so that we can tag a vlan

- However if you just want to trunk a normal linux bridge by 
  detault trunks to the vm

- libvirt also has portgroups concept but since cockpit machine
  web ui doesnt provide access to this functionaly, we will
  set the VLAN settings directly and vm/guest/domain XML config
  (as seen in below sections)


## Installed and started openvswitch
- Installed centos-release-nfv-openvswitch to gain access to 
  openvswitch
    - As a result, new dnf repo: "centos-nfv-openvswitch"
- Installed openvswith 3.5 and test tools

    ```
    dnf install centos-release-nfv-openvswitch
    dnf install openvswitch3.5.x86_64 openvswitch3.5-test.noarch
    systemctl enable --now  openvswitch.service
    ```


## Create ovs bridge and enslave 10Gbps nic (enp3s0)

- ovs-vsctl config changes are persistent. No need NM to help

```
    ovs-vsctl add-br ovs-br0
    ip addr show
    # Taking note there was an existing ovs-system bridge in addtion
    # to ovs-br0 that i am creating 
    
    #enslave enp3s0 (10Gbps) port that is trunked (faikam configured)
    ovs-vsctl add-port ovs-br0 enp3s0
    ovs-vsctl list-ports ovs-br0

    
    # When viewing in ip addr show, it will show that enp3s0 is enslaved
    # to ovs-system. However ovs-csctl list-ports ovs-br0 shows the 
    # ovs config
```

```
# ovs-vsctl show
dd086883-6ca3-48e4-8c1d-428c1104094d
    Bridge ovs-br0
        Port ovs-br0
            Interface ovs-br0
                type: internal
        Port enp3s0
            Interface enp3s0
    ovs_version: "3.5.1-6.el10s"
```
## Defining the libvirt network that uses OVS bridge/switch

```
cat <<EOF > libvirt-ovs-network-definition.xml
<network>
  <name>ovs-network</name>
  <forward mode='bridge'/>
  <bridge name='ovs-br0'/>
  <virtualport type='openvswitch'/>
</network>
EOF

virsh net-define libvirt-ovs-network-definition.xml
virsh net-start ovs-network
virsh net-autostart ovs-network

```

## Create a VM that uses the libvirt ovs-network
- The VM nic is trunked (ie it receives tagged/untagged traffic
- ```tcpdump``` command shows that the VM receives tagged traffic
  (if trunked to the vm)
- From IP perspective, it seems to get the DHCP from vlan 1 
  (likely due to switch config or maybe just the first DHCP reply it got)
    - VM can get IP
    - in terms of getting IP config from dhcp

> **THEREFORE** if the VM is connected to the 'ovs-network' defined
  the VM is by default trunked (tagged/untagged frames are 
  forwared to the VM. This is good for pfsense

```shell
tcpdump -i NIC -s 0 -e -nn vlan 
```
- Running the above ```tcpdump``` **INSIDE** the vm shows it can 
  see vlan traffic (ie trunk working)

## Edit the VM XML network settings to set to a particular vlan
- VMs with network set to 'ovs-network' are trunked by default
- Add the <vlan > ... </vlan> section to the <interface> section
  (if you want the VM receive traffic from only a certain VLAN
  and to be unaware of the VLAN id.)
- Stop and start vm
- Based on testing works.
- Only small issue, cockpit-machine keeps reporting pending
  changes even if vm was stopped and started and the changes
  are already effective

```
virsh edit --domain VM
```

- Set/Verify ```type='network'```
    - As we have defined a libvirt network called ovs-network
      that in turn is configured in bridge mode

- Set/Verify ```<source network='ovs-network' />``` which is the 
  network defined at libvirt level which is bridged to the ovs switch

- Set ```<vlan><tag id='XX' /></vlan>```
- Refer to libvirt domain xml docs for other examples

```xml
    <interface type='network'> 
      <mac address='52:54:00:02:07:6a'/> 
      <source network='ovs-network'/> 
      <vlan> 
        <tag id='11'/> 
      </vlan> 
      <model type='virtio'/> 
      <address type='pci' domain='0x0000' bus='0x01' slot='0x00' function='0x0'/> 
    </interface> 
```

# Setting host machine to participate in a specific VLAN

```shell
# Set the ovs-br0 internal port to be tagged as 11
# The ovs-br0 internal port is mapped to interface ovs-br0
# We will set ip at ovs-br0
ovs-vsctl set port ovs-br0 tag=11

# ovs-vsctl  show
dd086883-6ca3-48e4-8c1d-428c1104094d
    Bridge ovs-br0
        Port ovs-br0
            tag: 11
            Interface ovs-br0
                type: internal
        Port vnet0
            Interface vnet0
        Port enp3s0
            Interface enp3s0
    ovs_version: "3.5.1-6.el10s"

```

> ovs-br0 (ovs bridge) has a **port** called ovs-br0 which is attached
  to the interface ovs-br0. We tag the ovs port as vlan 11 making
  it an access port

> Because we did not use NetworkManagger to create the ovs bridge
  we are limited to use ip command and dhcpd commands to set 
  the ip on ovs-bro interface non-persitentlty

```shell
# To get ip from vlan 11 dhcpd server
dhcpcd -n ovs-br0

# to terminate dhcp lease
dhcpcd -k ovs-br0

# For static ip
ip addr add 10.1.1.57/24 dev  ovs-br0
ip link set dev ovs-br0 up
```

# Testing back linux bridge method as difficulty in persisting IP for ovs-br0

- Use cockpit network section

1. Add **br-trunk** bridge with enp3s0 (trunk physical interface) as member
    - Disable ipv4, ipv4 on the br-trunk connection
    - Enable STP (nmcli when adding a bridge enables STP by default)
2. Add a VLAN where parent is **br-trunk** with vlan id eg 11
    - Disable ipv4, ipv6 on the connection
    - Therefore we are adding an access  port to br-trunk for vlan 11
3. Add **br11** bridge which is connected to the vlan11 port on br-trunk
    - Choose the vlan port created as member of this bridge
    - enable STP

4. Update br11 to enable IPV4.method auto -- if theory workss,
   gets IP from vlan11 dhcp server

TO TEST whether this config works:
    VM - bridge to LAN - br-trunk ---> VM sees the tagged traffic
    VM - bridge to LAN - br11     ---> VM sees untagged traffic for vlan11

IF this config works, i think we skip ovs, as it will be simpler
for day to day management.
- **TESTED** , this config works. So no loner using OVS. Just linux briges and vlans
  created from cockpit web ui. This results in easier management

# VLAN Info for Midvalley (KL1&2)
```
11    - office
21-27 - KL1
31-39 - KL2
42    - Exam (KL2 exam)
51-52 - Wifi (KL1-KL2)
66    - lab (eg my vmware server)
500   - Unifi
600   - Times
```
