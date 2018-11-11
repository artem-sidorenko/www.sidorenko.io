+++
title = "OpenStack Networking: Open vSwitch and VXLAN introduction"
date = "2018-11-08T12:30:00+01:00"
tags = ["openstack", "linux", "network"]
+++

In my new job I had to deploy a new [OpenStack] environment. It was about 5 years ago when I did last time anything related to the OpenStack infrastructure deployment. Unused knowledge vanishes pretty fast, besides that OpenStack is developing. So I had to refresh a lot of things and even to build knowledge from scratch in some areas.

From my point of view OpenStack networking is one of the most complicated parts if you want to do it right. This series of posts aims to give a brief introduction to this topic. It will cover following setup:

- usage of common IP network as a transport network for overlay networks (virtualised networks of VMs)
- usage of [Open vSwitch] as technology for overlay networks and connectivity of VMs cross over different hypervisors
- usage of [VXLAN] for tunnelling of overlay network communication between different nodes
- dedicated control node, where all OpenStack API services are running
- dedicated network node, where L3 agent, metadata agent and dhcp agents are running
- examples are done on the RHEL/CentOS 7 Linux distribution

OpenStack uses Open vSwitch and VXLAN heavily in this setup, so I start first with introduction to Open vSwitch and VXLAN in this post. OpenStack networking architecture and implementation will be covered in the further posts.

<!--more-->

# Technical playground

I prepared a technical playground, so we can try out the things with Open vSwitch. You should have [VirtualBox] and [Vagrant] installed for that.

Please save this `Vagrantfile` on your system and invoke `vagrant up`. This will give you a virtualised environment with two CentOS VMs, which have Open vSwitch installed and running and have a private network connection between them.

```ruby
# Vagrantfile

Vagrant.configure('2') do |config|
  config.vm.box = 'bento/centos-7.5'

  config.vm.provision "shell", inline: <<~SHELL
    yum install -y openvswitch
    systemctl start openvswitch
    systemctl enable openvswitch
  SHELL

  config.vm.define 'node1' do |node|
    node.vm.hostname = 'node1'
    node.vm.network 'private_network', ip: '192.168.50.11'
  end

  config.vm.define 'node2' do |node|
    node.vm.hostname = 'node2'
    node.vm.network 'private_network', ip: '192.168.50.12'
  end
end
```

Login to the nodes and show the Open vSwitch configuration:

```shell
# it takes some time
$ vagrant up
....
$ vagrant ssh node1
[vagrant@node1 ~]$ sudo -i bash
[root@node1 ~]# ovs-vsctl show
9deaaff2-d6ce-4e05-9b99-0c9cf59efd53
    ovs_version: "2.0.0"
[root@node1 ~]# exit
exit
[vagrant@node1 ~]$ exit
logout
Connection to 127.0.0.1 closed.
$ vagrant ssh node2
[vagrant@node2 ~]$
[vagrant@node2 ~]$ sudo -i bash
[root@node2 ~]# ovs-vsctl show
028a296c-6e1f-4347-b268-395d10801e42
    ovs_version: "2.0.0"
[root@node2 ~]# exit
exit
[vagrant@node2 ~]$ exit
logout
Connection to 127.0.0.1 closed.
```

# Open vSwitch introduction

Maybe you are already familiar with [Linux bridges], where you can create a layer 2 bridge of several interfaces so it behaves in the end like a switched network. [Open vSwitch] goes several steps further: it's an open source mature switch implementation, which runs on Linux. You can create multiple switches, VLANs, make interconnections, plug interfaces of your KVM VMs. One of most important features of Open vSwitch is: it's a distributed switch. It means: you can run several VMs on several KVM hypervisor nodes and have Open vSwitch connecting them together:

![openvswitch overview](overview-openvswitch.png)

In this post I focus on the practical side of Open vSwitch usage and I'm not going to cover Open vSwitch architecture or its internals. You can find good summaries and insights in [this post series by Arthur Chiao][OVS Deep Dive 1: vswitchd].

Let's create some network bridges and interfaces, which simulate the VMs and virtual networks in our playground setup. [Veth interfaces] are used here. Each veth pair represent ethernet devices, which are connected together. You can configure IP on one interface and plug another one to the switch, so it looks like a virtual cable.

Our desired setup:

![openvswitch playground setup](openvswitch-playground-setup.png)

```shell
# node1
# create and configure the first vmA1 interface
[root@node1 ~]# ip link add dev vmA1-sw type veth peer name vmA1
[root@node1 ~]# ip link set vmA1-sw up
[root@node1 ~]# ip link set vmA1 up
[root@node1 ~]# ip addr add 192.168.60.11/24 dev vmA1

# create and configure the second vmB1 interface
[root@node1 ~]# ip link add dev vmB1-sw type veth peer name vmB1
[root@node1 ~]# ip link set vmB1-sw up
[root@node1 ~]# ip link set vmB1 up
[root@node1 ~]# ip addr add 192.168.70.11/24 dev vmB1

# create tenantA and tenantB bridges, they represent the virtual switches
[root@node1 ~]# ovs-vsctl add-br tenantA
[root@node1 ~]# ovs-vsctl add-br tenantB

# plugin the simulated interfaces to the bridges
[root@node1 ~]# ovs-vsctl add-port tenantA vmA1-sw
[root@node1 ~]# ovs-vsctl add-port tenantB vmB1-sw

[root@node1 ~]# ovs-vsctl show
30605565-4210-4ce6-bbc8-2356ece5f7ce
    Bridge tenantB
        Port tenantB
            Interface tenantB
                type: internal
        Port "vmB1-sw"
            Interface "vmB1-sw"
    Bridge tenantA
        Port tenantA
            Interface tenantA
                type: internal
        Port "vmA1-sw"
            Interface "vmA1-sw"
    ovs_version: "2.0.0"

# node2
# create and configure the first vmA2 interface
[root@node2 ~]# ip link add dev vmA2-sw type veth peer name vmA2
[root@node2 ~]# ip link set vmA2-sw up
[root@node2 ~]# ip link set vmA2 up
[root@node2 ~]# ip addr add 192.168.60.12/24 dev vmA2

# create and configure the second vmB2 interface
[root@node2 ~]# ip link add dev vmB2-sw type veth peer name vmB2
[root@node2 ~]# ip link set vmB2-sw up
[root@node2 ~]# ip link set vmB2 up
[root@node2 ~]# ip addr add 192.168.70.12/24 dev vmB2

# create tenantA and tenantB bridges, they represent the virtual switches
[root@node2 ~]# ovs-vsctl add-br tenantA
[root@node2 ~]# ovs-vsctl add-br tenantB

# plugin the simulated interfaces to the bridges
[root@node2 ~]# ovs-vsctl add-port tenantA vmA2-sw
[root@node2 ~]# ovs-vsctl add-port tenantB vmB2-sw

[root@node2 ~]# ovs-vsctl show
881c2467-5d6d-458e-ae74-6c017ef9b369
    Bridge tenantB
        Port tenantB
            Interface tenantB
                type: internal
        Port "vmB2-sw"
            Interface "vmB2-sw"
    Bridge tenantA
        Port "vmA2-sw"
            Interface "vmA2-sw"
        Port tenantA
            Interface tenantA
                type: internal
    ovs_version: "2.0.0"

# try to ping the simulated interfaces
[root@node2 ~]# ping -c 2 192.168.60.11
PING 192.168.60.11 (192.168.60.11) 56(84) bytes of data.
From 192.168.60.12 icmp_seq=1 Destination Host Unreachable
From 192.168.60.12 icmp_seq=2 Destination Host Unreachable

--- 192.168.60.11 ping statistics ---
2 packets transmitted, 0 received, +2 errors, 100% packet loss, time 999ms
pipe 2
[root@node2 ~]# ping -c 2 192.168.70.11
PING 192.168.70.11 (192.168.70.11) 56(84) bytes of data.
From 192.168.70.12 icmp_seq=1 Destination Host Unreachable
From 192.168.70.12 icmp_seq=2 Destination Host Unreachable

--- 192.168.70.11 ping statistics ---
2 packets transmitted, 0 received, +2 errors, 100% packet loss, time 999ms
pipe 2
```

Ping does not work as there is no connection between the bridges on the nodes yet.

# VXLAN and GRE

In order to pass traffic of virtual tenant networks between nodes it should be encapsulated and transferred via existing IP network on the physical connection. Common Open vSwitch deployments use [VXLAN] or [GRE] protocols for this purpose.

VXLAN and GRE are pretty similar in this particular use case, their tasks are:

- encapsulation of traffic of virtual networks
- differentiation between different virtual networks

I decided to use VXLAN as a more modern implementation then GRE,  mostly because of existing Layer-4 port number. This gives for me a better distribution over 10GE port channels on the hypervisors and physical network infrastructure. I was also lucky to have network cards ([Intel 82599ES](https://ark.intel.com/products/41282/Intel-82599ES-10-Gigabit-Ethernet-Controller)) with support for VXLAN and GRE offloading.

Lets compare a typical ssh packet with ssh packet, which was sent via VXLAN in some virtualised network:

![ssh packet](without-vxlan.png)


![ssh packet on some virtualised network](vxlan.png)

On the second picture we see much more layers, if you take a detailed look - you will recognise the VXLAN encapsulation of SSH packet and VXLAN communication over another network below. This is basically the reason for the term `overlay networks`: virtualised networks are built over another network layer.

Lets have a detailed look to the VXLAN layer:

![detailed view on VXLAN layer](vxlan-detailed.png)

You can see the VNI - VXLAN Network Identifier header. This header is responsible for differentiation between different virtual networks.

Go ahead and create VXLAN tunnels between our VMs (you might be surprised about `--` command syntax, I'll cover it later): 

```shell
# node1
[root@node1 ~]# ovs-vsctl add-port tenantA vxlanA -- set interface vxlanA type=vxlan options:remote_ip=192.168.50.12 options:key=5000
[root@node1 ~]# ovs-vsctl add-port tenantB vxlanB -- set interface vxlanB type=vxlan options:remote_ip=192.168.50.12 options:key=6000
[root@node1 ~]# ovs-vsctl show
30605565-4210-4ce6-bbc8-2356ece5f7ce
    Bridge tenantB
        Port tenantB
            Interface tenantB
                type: internal
        Port vxlanB
            Interface vxlanB
                type: vxlan
                options: {key="6000", remote_ip="192.168.50.12"}
        Port "vmB1-sw"
            Interface "vmB1-sw"
    Bridge tenantA
        Port tenantA
            Interface tenantA
                type: internal
        Port "vmA1-sw"
            Interface "vmA1-sw"
        Port vxlanA
            Interface vxlanA
                type: vxlan
                options: {key="5000", remote_ip="192.168.50.12"}
    ovs_version: "2.0.0"

# node2
[root@node2 ~]# ovs-vsctl add-port tenantA vxlanA -- set interface vxlanA type=vxlan options:remote_ip=192.168.50.11 options:key=5000
[root@node2 ~]# ovs-vsctl add-port tenantB vxlanB -- set interface vxlanB type=vxlan options:remote_ip=192.168.50.11 options:key=6000

[root@node2 ~]# ovs-vsctl show
881c2467-5d6d-458e-ae74-6c017ef9b369
    Bridge tenantB
        Port vxlanB
            Interface vxlanB
                type: vxlan
                options: {key="6000", remote_ip="192.168.50.11"}
        Port tenantB
            Interface tenantB
                type: internal
        Port "vmB2-sw"
            Interface "vmB2-sw"
    Bridge tenantA
        Port "vmA2-sw"
            Interface "vmA2-sw"
        Port vxlanA
            Interface vxlanA
                type: vxlan
                options: {key="5000", remote_ip="192.168.50.11"}
        Port tenantA
            Interface tenantA
                type: internal
    ovs_version: "2.0.0"
```

Now run the ping - it works!

```shell
[root@node2 ~]# ping -c 2 192.168.60.11
PING 192.168.60.11 (192.168.60.11) 56(84) bytes of data.
64 bytes from 192.168.60.11: icmp_seq=1 ttl=64 time=0.466 ms
64 bytes from 192.168.60.11: icmp_seq=2 ttl=64 time=0.911 ms

--- 192.168.60.11 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1000ms
rtt min/avg/max/mdev = 0.466/0.688/0.911/0.224 ms
[root@node2 ~]# ping -c 2 192.168.70.11
PING 192.168.70.11 (192.168.70.11) 56(84) bytes of data.
64 bytes from 192.168.70.11: icmp_seq=1 ttl=64 time=1.69 ms
64 bytes from 192.168.70.11: icmp_seq=2 ttl=64 time=0.673 ms

--- 192.168.70.11 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 0.673/1.184/1.695/0.511 ms
```

# Open vSwitch control commands

Now I would like to give a short summary over Open vSwitch control commands. `ovs-vsctl` is one of the important commands, which allows the configuration of Open vSwitch. `ovs-vsctl` talks to `ovsdb-server` process, which maintains the Open vSwitch configuration database.

`ovs-vsctl` can take a single or multiple commands per call. If multiple commands are given, they should be separated by `--` (like we did it below, where we created a new interface and configured its parameters in one call).

You can see the list of possible commands by invoking `ovs-vsctl --help`. If you create a new interface with `ovs-vsctl add-port`, this interface can be from different types and even take different options (see VXLAN interface example above). Documentation of this options and interface types is located in the man page of Open vSwitch database scheme: `man 5 ovs-vswitchd.conf.db`. Another possible way to display possible options is via `ovs-vsctl list` command. This command works on the database layer of Open vSwitch and shows possible attributes for some specific entity:

```shell
# display DB information for existing bridges
[root@node1 ~]# ovs-vsctl list bridge
_uuid               : 8da8a6fc-fda2-4811-b109-389d2c189030
controller          : []
datapath_id         : "0000fea6a88d1148"
datapath_type       : ""
external_ids        : {}
fail_mode           : []
flood_vlans         : []
flow_tables         : {}
ipfix               : []
mirrors             : []
name                : tenantA
netflow             : []
other_config        : {}
ports               : [0743963c-bd76-4b7f-a9d3-cfd015d46975, 53b10a77-caf4-4438-92fb-51bfe796e7d3, 6f21f978-14ef-4684-a2d7-ee3032b09b4b, b510c5a9-cdac-468e-bbb0-45ed0860b7da]
protocols           : []
sflow               : []
status              : {}
stp_enable          : false

_uuid               : 79b0b2e5-c9b5-4b0c-ad4a-7c3b87855b93
controller          : []
datapath_id         : "0000e6b2b0790c4b"
datapath_type       : ""
external_ids        : {}
fail_mode           : []
flood_vlans         : []
flow_tables         : {}
ipfix               : []
mirrors             : []
name                : tenantB
netflow             : []
other_config        : {}
ports               : [45025cd6-4edc-4da9-98e3-482b1ec08522, a16e288a-8ffa-4938-ade6-0214a991e8b1, fbb77350-255b-4bf4-bf67-f8574173878e]
protocols           : []
sflow               : []
status              : {}
stp_enable          : false

# select only the bridge tenantA
[root@node1 ~]# ovs-vsctl list bridge tenantA
_uuid               : 8da8a6fc-fda2-4811-b109-389d2c189030
controller          : []
datapath_id         : "0000fea6a88d1148"
datapath_type       : ""
external_ids        : {}
fail_mode           : []
flood_vlans         : []
flow_tables         : {}
ipfix               : []
mirrors             : []
name                : tenantA
netflow             : []
other_config        : {}
ports               : [0743963c-bd76-4b7f-a9d3-cfd015d46975, 53b10a77-caf4-4438-92fb-51bfe796e7d3, 6f21f978-14ef-4684-a2d7-ee3032b09b4b, b510c5a9-cdac-468e-bbb0-45ed0860b7da]
protocols           : []
sflow               : []
status              : {}
stp_enable          : false
```

You can do the same thing for other entity types: ports, interfaces:

```shell
[root@node1 ~]# ovs-vsctl list interface
...
[root@node1 ~]# ovs-vsctl list port
...
```

# Open vSwitch ports

Open vSwitch supports different port types. Let's have a look to the common types.

## Layer 2 and Layer 3 ports

Typical switch ports are layer 2 ports, traffic gets forwarded by the switch between this ports. This ports do not have any layer 3 IP configuration. Even with linux bridge you can observe  this behaviour: if you have IP configuration on eth0 and you add it to some bridge - you will lose the connection on eth0 as eth0 starts to work as layer 2 port only. (Output of `ifconfig` is a bit confusing in a such situation, because you still see the IP configuration on the interface) Solution for linux bridge setup is to move the IP configuration from eth0 to the bridge interface (e.g. br0).

You can observe the similar thing with Open vSwitch, as its still the switch, even if a powerful one. Open vSwitch offers a solution with `internal` port type. `Internal` port is a layer 3 port, which also gets exposed outside of Open vSwitch, so you can make the IP configuration on this interface.

Lets try it out and create a new internal port:

```shell
[root@node1 ~]# ovs-vsctl add-port tenantA internalPort -- set interface internalPort type=internal
[root@node1 ~]# ip addr add 192.168.60.50/24 dev internalPort
[root@node1 ~]# ip link set internalPort up

# now try to ping it from node2
[root@node2 ~]# ping -c2 192.168.60.50
PING 192.168.60.50 (192.168.60.50) 56(84) bytes of data.
64 bytes from 192.168.60.50: icmp_seq=1 ttl=64 time=0.882 ms
64 bytes from 192.168.60.50: icmp_seq=2 ttl=64 time=0.937 ms

--- 192.168.60.50 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 0.882/0.909/0.937/0.040 ms
```

Now have a look to the existing bridges. You can see, when you create a new bridge - it automatically gets an internal port with the same name like bridge name:

```shell
[root@node1 ~]# ovs-vsctl show
30605565-4210-4ce6-bbc8-2356ece5f7ce
    Bridge tenantB
        Port tenantB
            Interface tenantB
                type: internal
        Port vxlanB
            Interface vxlanB
                type: vxlan
                options: {key="6000", remote_ip="192.168.50.12"}
        Port "vmB1-sw"
            Interface "vmB1-sw"
    Bridge tenantA
        Port internalPort
            Interface internalPort
                type: internal
        Port tenantA
            Interface tenantA
                type: internal
        Port "vmA1-sw"
            Interface "vmA1-sw"
        Port vxlanA
            Interface vxlanA
                type: vxlan
                options: {key="5000", remote_ip="192.168.50.12"}
    ovs_version: "2.0.0"
[root@node1 ~]# ip link | grep tenant
5: tenantA: <BROADCAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
6: tenantB: <BROADCAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
``` 

You can find much more details in the [post of Arthur Chiao](https://arthurchiao.github.io/blog/ovs-deep-dive-6-internal-port/).

## Mirror port

Maybe you already know the `mirror` or `span` ports from hardware switches. This ports are useful for mirroring of specific traffic for troubleshooting purposes. Open vSwitch supports mirror ports too:

```shell
# create internal mirror port, we need an interface where we can start wireshark/tcpdump on
[root@node1 ~]# ovs-vsctl add-port tenantA mirrorPort -- set interface mirrorPort type=internal
# create mirror configuration
[root@node1 ~]# ovs-vsctl --id=@vmA1-sw get port vmA1-sw --\
                          --id=@mirrorPort get port mirrorPort --\
                          --id=@mirror create mirror name=mirror \
                          select-dst-port=@vmA1-sw select-src-port=@vmA1-sw output-port=@mirrorPort --\
                          set bridge tenantA mirrors=@mirror

# lets install tshark
[root@node1 ~]# yum -y install wireshark
...

# and now invoke ping 192.168.60.11 on the node2 in parallel
[root@node2 ~]# ping 192.168.60.11
...

# start wireshark on the mirror port
[root@node1 ~]# tshark -c 6 -i mirrorPort
Running as user "root" and group "root". This could be dangerous.
Capturing on 'mirrorPort'
  1 0.000000000 192.168.60.12 -> 192.168.60.11 ICMP 98 Echo (ping) request  id=0x1986, seq=1/256, ttl=64
  2 0.000259634 192.168.60.11 -> 192.168.60.12 ICMP 98 Echo (ping) reply    id=0x1986, seq=1/256, ttl=64 (request in 1)
  3 1.000990898 192.168.60.12 -> 192.168.60.11 ICMP 98 Echo (ping) request  id=0x1986, seq=2/512, ttl=64
  4 1.001016624 192.168.60.11 -> 192.168.60.12 ICMP 98 Echo (ping) reply    id=0x1986, seq=2/512, ttl=64 (request in 3)
  5 2.000872404 192.168.60.12 -> 192.168.60.11 ICMP 98 Echo (ping) request  id=0x1986, seq=3/768, ttl=64
  6 2.000898423 192.168.60.11 -> 192.168.60.12 ICMP 98 Echo (ping) reply    id=0x1986, seq=3/768, ttl=64 (request in 5)
6 packets captured

# cleanup the mirror configuration
[root@node1 ~]# ovs-vsctl clear bridge tenantA mirrors
[root@node1 ~]# ovs-vsctl del-port mirrorPort
```

The `--id` commands within mirror configuration provide aliases to the particular interfaces, mirror configuration command requires interface IDs and not names. There are different options, which can be specified within `create mirror` command. Important options from `man 5 ovs-vswitch.conf.db`:

```
...
     Selecting Packets for Mirroring:
       To  be  selected  for  mirroring,  a  given packet must enter or leave the bridge through a
       selected port and it must also be in one of the selected VLANs.

       select_all: boolean
              If true, every packet arriving or departing on any port is selected for mirroring.

       select_dst_port: set of weak reference to Ports
              Ports on which departing packets are selected for mirroring.

       select_src_port: set of weak reference to Ports
              Ports on which arriving packets are selected for mirroring.

       select_vlan: set of up to 4,096 integers, in range 0 to 4,095
              VLANs on which packets are selected for mirroring.  An empty set selects packets  on
              all VLANs.
              
   Mirroring Destination Configuration:
       These columns are mutually exclusive.  Exactly one of them must be nonempty.

       output_port: optional weak reference to Port
              Output port for selected packets, if nonempty.

              Specifying  a  port  for mirror output reserves that port exclusively for mirroring.
              No frames other than those selected for mirroring via this column will be  forwarded
              to the port, and any frames received on the port will be discarded.

              The  output  port may be any kind of port supported by Open vSwitch.  It may be, for
              example, a physical port (sometimes called SPAN) or a GRE tunnel.
...
```

Please keep in mind, you can't mirror patch ports (see [this post from Arthur Chiao][OVS Deep Dive 4: OVS netdev and Patch Port] for more details). If you need to mirror traffic of patch ports, please use veth pairs instead.

## Patch port

Patch ports are similar to the physical cable interconnecting two switches (or to the veth pair plugged into both Open vSwitch bridges). 

Let's introduce a new bridge `tenantC` with one internal port and connect this bridge with `tenantA` bridge via patch ports together. In this setup it should be possible to reach tenantC internal port IP addresses between node1 and node2 because of existing patch connection - the inter-node communication goes through `tenantA` bridge and its VXLAN interconnection.

![patch port overview](openvswitch-patch-port.png)

```shell
# create teneantC bridge and configure tenantC internal port
[root@node1 ~]# ovs-vsctl add-br tenantC
[root@node1 ~]# ip addr add 192.168.80.11/24 dev tenantC
[root@node1 ~]# ip link set tenantC up

[root@node2 ~]# ovs-vsctl add-br tenantC
[root@node2 ~]# ip addr add 192.168.80.12/24 dev tenantC
[root@node2 ~]# ip link set tenantC up

# not try to reach 192.168.80.11 from node2, it should fail
# as do not have the patch links yet in place
[root@node2 ~]# ping -c2 192.168.80.11
PING 192.168.80.11 (192.168.80.11) 56(84) bytes of data.
From 192.168.80.12 icmp_seq=1 Destination Host Unreachable
From 192.168.80.12 icmp_seq=2 Destination Host Unreachable

--- 192.168.80.11 ping statistics ---
2 packets transmitted, 0 received, +2 errors, 100% packet loss, time 999ms
pipe 2

# establish the patch connections
[root@node1 ~]# ovs-vsctl add-port tenantC patchC --\
                          add-port tenantA patchA --\
                          set interface patchC type=patch options:peer=patchA --\
                          set interface patchA type=patch options:peer=patchC
                          
[root@node2 ~]# ovs-vsctl add-port tenantC patchC --\
                          add-port tenantA patchA --\
                          set interface patchC type=patch options:peer=patchA --\
                          set interface patchA type=patch options:peer=patchC

# try the ping again
[root@node2 ~]# ping -c2 192.168.80.11
PING 192.168.80.11 (192.168.80.11) 56(84) bytes of data.
64 bytes from 192.168.80.11: icmp_seq=1 ttl=64 time=1.08 ms
64 bytes from 192.168.80.11: icmp_seq=2 ttl=64 time=0.499 ms

--- 192.168.80.11 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1000ms
rtt min/avg/max/mdev = 0.499/0.792/1.085/0.293 ms
```

If you are interested in implementation details of patch ports, have a look at the [great post by Arthur Chiao][OVS Deep Dive 4: OVS netdev and Patch Port].

## VLANs

Similar to the usual managed switch, you can use VLANs with Open vSwitch too.

Lets create some additional internal ports on the `tenantC` switch with VLAN tags:

```shell
[root@node1 ~]# ovs-vsctl add-port tenantC vlanPortC tag=10 --\
                set interface vlanPortC type=internal
[root@node1 ~]# ip addr add 192.168.90.11/24 dev vlanPortC
[root@node1 ~]# ip link set vlanPortC up

[root@node2 ~]# ovs-vsctl add-port tenantC vlanPortC tag=10 --\
                set interface vlanPortC type=internal
[root@node2 ~]# ip addr add 192.168.90.12/24 dev vlanPortC
[root@node2 ~]# ip link set vlanPortC up

[root@node1 ~]# ovs-vsctl show
30605565-4210-4ce6-bbc8-2356ece5f7ce
    Bridge tenantB
        Port tenantB
            Interface tenantB
                type: internal
        Port vxlanB
            Interface vxlanB
                type: vxlan
                options: {key="6000", remote_ip="192.168.50.12"}
        Port "vmB1-sw"
            Interface "vmB1-sw"
    Bridge tenantA
        Port internalPort
            Interface internalPort
                type: internal
        Port tenantA
            Interface tenantA
                type: internal
        Port "vmA1-sw"
            Interface "vmA1-sw"
        Port vxlanA
            Interface vxlanA
                type: vxlan
                options: {key="5000", remote_ip="192.168.50.12"}
        Port patchA
            Interface patchA
                type: patch
                options: {peer=patchC}
    Bridge tenantC
        Port vlanPortC
            tag: 10
            Interface vlanPortC
                type: internal
        Port tenantC
            Interface tenantC
                type: internal
        Port patchC
            Interface patchC
                type: patch
                options: {peer=patchA}
    ovs_version: "2.0.0"

[root@node2 ~]# ping -c2 192.168.90.11
PING 192.168.90.11 (192.168.90.11) 56(84) bytes of data.
64 bytes from 192.168.90.11: icmp_seq=1 ttl=64 time=1.21 ms
64 bytes from 192.168.90.11: icmp_seq=2 ttl=64 time=0.499 ms

--- 192.168.90.11 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 0.499/0.858/1.218/0.360 ms
```

If you would capture the traffic on the physical link and open it in wireshark, you could see the VLAN tag:

![vlan tag](vlan-tag.png)

## VXLAN and GRE ports

Some further ports types are tunnelling ports. In this post I cover only VXLAN and GRE, however there is also some further technologies like [Geneve] or [GREoIPsec].

You already configured VXLAN tunnelling above, GRE configuration way is almost the same. Let's remove the patch connections between `tenantC` and `tenantA`, GRE tunnel will do the interconnection between `tenantC` bridges.

```shell
# remove the patch ports
[root@node1 ~]# ovs-vsctl del-port patchC
[root@node1 ~]# ovs-vsctl del-port patchA

[root@node2 ~]# ovs-vsctl del-port patchC
[root@node2 ~]# ovs-vsctl del-port patchA

# ensure we lost the connectivity
[root@node2 ~]# ping -c2 192.168.90.11
PING 192.168.90.11 (192.168.90.11) 56(84) bytes of data.
From 192.168.90.12 icmp_seq=1 Destination Host Unreachable
From 192.168.90.12 icmp_seq=2 Destination Host Unreachable

--- 192.168.90.11 ping statistics ---
2 packets transmitted, 0 received, +2 errors, 100% packet loss, time 1000ms
pipe 2


# add the GRE tunnels
[root@node1 ~]# ovs-vsctl add-port tenantC greC -- set interface greC type=gre options:remote_ip=192.168.50.12

[root@node2 ~]# ovs-vsctl add-port tenantC greC -- set interface greC type=gre options:remote_ip=192.168.50.11

# ping again
[root@node2 ~]# ping -c2 192.168.90.11
PING 192.168.90.11 (192.168.90.11) 56(84) bytes of data.
64 bytes from 192.168.90.11: icmp_seq=1 ttl=64 time=2.63 ms
64 bytes from 192.168.90.11: icmp_seq=2 ttl=64 time=0.721 ms

--- 192.168.90.11 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 0.721/1.679/2.638/0.959 ms
```

If you would capture the traffic on the physical link and open it in wireshark, you could see the GRE header:

![GRE header](gre.png)

# MTU problem

As we use encapsulation, we get the common MTU problem which occurs in such cases. Ethernet has so called MTU - Maximum Transmission Unit. MTU defines the maximum size of ethernet frame, which can be transmitted over the line. When MTU size is exceeded, your IP package is splitted to the multiple packets or even gets dropped. In the VXLAN setup with encapsulation, we have Ethernet frames on the underlaying network and we also have ethernet frame on the overlay network. In this case we run into the common problem of MTU size: the frame on the underlaying network is bigger that standard MTU of `1500`. You can see the resulting problem on the picture below:

![MTU problem](mtu-problem.png)

In order to avoid any problems, we have to adjust MTU on the underlaying network. If we would do the calculation of additional headers (VXLAN + UDP + IP + Ethernet) we would get `1554` MTU size for typical IPv4 traffic.

![MTU 1554](mtu-fix.png)

See also [MTU Considerations for VXLAN] for more details.

You can also consider to use [Jumbo frames] with MTU of `9000`.

<br/><br/>

In the next post I'm going to cover OpenFlow with Open vSwitch and then switch to the question how OpenStack uses this underlying technologies for its networking implementation.

# See too

- [What Is Open vSwitch?](http://docs.openvswitch.org/en/latest/intro/what-is-ovs/)
- [Open vSwitch FAQ - Basic configuration](https://docs.openvswitch.org/en/latest/faq/configuration/)
- [Open vSwitch Advanced Features](http://docs.openvswitch.org/en/latest/tutorials/ovs-advanced/)
- [Open vSwitch Cheat Sheet](https://therandomsecurityguy.com/openvswitch-cheat-sheet/)
- [Using GRE Tunnels with Open vSwitch](https://blog.scottlowe.org/2013/05/07/using-gre-tunnels-with-open-vswitch/)
- [OVS Deep Dive 1: vswitchd][OVS Deep Dive 1: vswitchd]
- [OVS Deep Dive 4: OVS netdev and Patch Port][OVS Deep Dive 4: OVS netdev and Patch Port]
- [OVS Deep Dive 6: Internal Port](https://arthurchiao.github.io/blog/ovs-deep-dive-6-internal-port/)
- [VXLAN vs GRE](https://www.reddit.com/r/networking/comments/4bko87/vxlan_vs_gre/)
- [Creating Overlay Networks Using Intel® Ethernet Converged Network Adapters](https://www.intel.com/content/dam/www/public/us/en/documents/technology-briefs/overlay-networks-using-converged-network-adapters-brief.pdf)
- [Optimizing the Virtual Network with VXLAN Overlay Offloading](https://software.intel.com/en-us/blogs/2015/01/29/optimizing-the-virtual-networks-with-vxlan-overlay-offloading)
- [Overlay Tunneling with Open vSwitch - GRETAP, VXLAN, Geneve, GREoIPsec][Overlay Tunneling with Open vSwitch - GRETAP, VXLAN, Geneve, GREoIPsec]
- [MTU Considerations for VXLAN][MTU Considerations for VXLAN]


[OpenStack]: https://www.openstack.org
[Open vSwitch]: http://www.openvswitch.org
[VXLAN]: https://en.wikipedia.org/wiki/Virtual_Extensible_LAN
[GRE]: https://en.wikipedia.org/wiki/Generic_Routing_Encapsulation
[Linux bridges]: https://wiki.debian.org/BridgeNetworkConnections
[VirtualBox]: https://www.virtualbox.org
[Vagrant]: https://www.vagrantup.com
[veth interfaces]: http://man7.org/linux/man-pages/man4/veth.4.html
[OVS Deep Dive 4: OVS netdev and Patch Port]: https://arthurchiao.github.io/blog/ovs-deep-dive-4-patch-port/
[Overlay Tunneling with Open vSwitch - GRETAP, VXLAN, Geneve, GREoIPsec]: https://costiser.ro/2016/07/07/overlay-tunneling-with-openvswitch-gre-vxlan-geneve-greoipsec/
[Geneve]: https://costiser.ro/2016/07/07/overlay-tunneling-with-openvswitch-gre-vxlan-geneve-greoipsec/#geneve
[GREoIPsec]: https://costiser.ro/2016/07/07/overlay-tunneling-with-openvswitch-gre-vxlan-geneve-greoipsec/#greoipsec
[OVS Deep Dive 1: vswitchd]: https://arthurchiao.github.io/blog/ovs-deep-dive-1-vswitchd/ 
[MTU Considerations for VXLAN]: https://keepingitclassless.net/2014/03/mtu-considerations-vxlan/
[Jumbo frames]: https://en.wikipedia.org/wiki/Jumbo_frame