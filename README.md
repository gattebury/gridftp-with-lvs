# GridFTP with LVS

Describes load balanced GridFTP setup as implemented at T2_US_Nebraska.

## Background Documentation

[Linux Virtual Server Website][1]: old and a bit crufty looking, but certainly a good starting point to understand what LVS is and the common modes of operation. The pages describing [IPVS][2], [direct routing][3], and [the ARP problem][4] are of the most interest here.

Red Hat's documentation for load balancing with LVS for [RHEL6][5] and [RHEL7][6] are excellent reads as well.

[1]:http://www.linuxvirtualserver.org/
[2]:http://kb.linuxvirtualserver.org/wiki/IPVS
[3]:http://www.linuxvirtualserver.org/VS-DRouting.html
[4]:http://kb.linuxvirtualserver.org/wiki/ARP_Issues_in_LVS/DR_and_LVS/TUN_Clusters
[5]:https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Load_Balancer_Administration/index.html
[6]:https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Load_Balancer_Administration/index.html

Try to understand the differences between terms and components. IPVS is part of the Linux kernel itself. Keepalived, HAProxy, Piranha, etc all offer different ways of automating / interacting with the IPVS layer. Some can also provide high-availability of the LVS layer itself, which is different from any failover or balancing that LVS is doing. There are other tools for all of this as well, and even commercial solutions.

tldr; this isn't the only way to do it.


## What's the purpose?

The overall goal is to provide a single endpoint for GridFTP transfers which is load balanced over multiple 'real' servers and, in theory, self-healing and more reliable than previous incarnations such as using BestMan.

An additional benefit is that you can add and remove servers on the fly without requiring downtime thus making maintenance work and upgrades that much easier.

## Components

The pieces of the puzzle as implemented at T2_US_Nebraska are as follows:

* 2x LVS director hosts. Low-end Dell R310s with 2.8GHz Pentium G6950 procs and 1Gb NIC. These could i386 calculators in all reality as they just mangle packets.
* 12x GridFTP transfer hosts. These are the realservers and in our case Dell R410s with a 10Gb NIC.
* Above hosts all live on the same L2 network segment.
* CentOS 7 on the above hosts. SL6 was used in the past with no problem.
* [arioch/puppet-keepalived][7] puppet module for keepalived.conf config generation
[7]:https://github.com/arioch/puppet-keepalived

**NOTE:** You can implement all of this with a single LVS director host and as few as one realserver. You can also maintain the LVS config(s) manually or via other utilities - we just happen to use [keepalived][8] and a puppet module for its config.

[8]:http://www.keepalived.org/

## Setup

### LVS Setup

You need at least one LVS director and how you configure it depends largely on your OS and what tools you wish to use. At Nebraska we used a manually generated _/etc/sysconfig/ha/lvs.cf_ in the past when we ran SL6 with Piranha. When we switched to CentOS 7 and Keepalived, we also started using puppet to generate the keepalived.conf via the [arioch/puppet-keepalived][7] module.

All the module parameters come from Hiera. The IPv4 GridFTP highlights from our configs are as follows with `129.93.239.157` being our virtual IP and `129.93.239.[133,134]` being the IPs of our two LVS directors. Note that you can also do IPv6, and have multiple services. Only showing IPv4 gridftp here for simplicity.

The VRRP instance which sets up the virtual IP and master/slave pair. Note that the source/peer IPs would be different for the other host's .yaml file.

~~~~
### arioch/puppet-keepalived VRRP
keepalived::vrrp_instances:

    # red-gridftp.unl.edu IPv4
    VI_1:
        interface: 'em1'
        state: 'MASTER'
        virtual_router_id: '1'
        priority: '100'
        virtual_ipaddress: '129.93.239.157/26'
        track_interface:
            - 'em1'
        unicast_source_ip: '129.93.239.133'
        unicast_peers: [ '129.93.239.134' ]
~~~~

The virtual server group which specifies the service IP/port and real server IP/port. Here we use the weighted least connection algorithm with direct routing and include a TCP_CHECK to remove failed/dead servers.

~~~~
### arioch/puppet-keepalived LVS
keepalived::virtual_servers:
    red-gridftp-v4:
        ip_address: '129.93.239.157'
        port: '2811'
        protocol: 'TCP'
        lb_algo: 'wlc'
        lb_kind: 'DR'
        tcp_check:
            connect_timeout: 3
        delay_loop: '10'
        real_servers:
            - ip_address: '129.93.239.184'
              port: '2811'
            - ip_address: '129.93.239.171'
              port: '2811'
            - ip_address: '129.93.239.173'
              port: '2811'
            - ip_address: '129.93.239.168'
              port: '2811'
            - ip_address: '129.93.239.168'
              port: '2811'
            - ip_address: '129.93.239.130'
              port: '2811'
            - ip_address: '129.93.239.136'
              port: '2811'
            - ip_address: '129.93.239.138'
              port: '2811'
            - ip_address: '129.93.239.178'
              port: '2811'
            - ip_address: '129.93.239.165'
              port: '2811'
            - ip_address: '129.93.239.187'
              port: '2811'
            - ip_address: '129.93.239.180'
              port: '2811'
~~~~

The [arioch/puppet-keepalived][7] puppet module generates the _/etc/keepalived/keepalived.conf_ config from all of this and maintains the keepalived.service. You can just as easily hand write a config and maintain that in whatever way you deem fit without needing puppet.

_/var/log/messages_ is your friend when starting and stopping keepalived, especially when monitoring the interaction between a "HA" pair of LVS directors.


### Realserver Setup

#### Solve the ARP problem

Multiple methods exist, but _arptables_ is easy to understand due to its similarity to _iptables_. It is perhaps more flexible as well than blocking ARP via sysctl.

In the Nebraska case, our virtual IPv4 address is `129.93.239.157` and we use the following to generate _/etc/syconfig/arptables_ on our realservers which all have _p1p1_ interfaces.
~~~~
arptables -F
arptables -A INPUT -d 129.93.239.157 -j DROP
arptables -A OUTPUT -s 129.93.239.157 -j mangle --mangle-ip-s `ifconfig p1p1 | sed -n 's/.*inet \([0-9.]\+\)\s.*/\1/p'`
arptables-save > /etc/sysconfig/arptables
~~~~

As with the _iptables_ service, you can enable the _arptables_ service and have this functionality persist across reboots.

#### Add secondary IPs as appropriate

The realservers need to have the virtual IP as alias / secondary. One can do this manually with:

~~~~
ip addr add 129.93.239.157/26 dev p1p1
ip -6 addr add 2600:900:6:1101::da7a:1055 dev lo
~~~~

Ensuring these aliases persist across reboots can be tricky, at least with RHEL7, as the network init scripts seem to not support it as in the past. Invoking the ancient _rc.local_ demon is one solution here.


#### IPv6 TCP_CHECK gotchas

Using global IPv6 addresses for realservers and TCP_CHECK will most likely not work as the director host will send the checks from the virtual IPv6 address (which is the most recently brought up). There are a few workarounds to this, but an easy one is to mark the virtual address as deprecated such that the checks originate from the real IPv6 address of the director as follows:

~~~
ip -6 addr change 2600:900:6:1101::da7a:1055/64 dev em1 preferred_lft 0
~~~

There are some mailing list entries pondering whether keepalived should deprecate the virtual address by default but I don't believe that happens in any version (there are patches, but ... effort). Using link-local addresses may be another workaround but I haven't tested that fully.


## Tips / tricks

* Watch _/var/log/messages_ to see what keepalived is doing, especially when it comes to the VRRP and heartbeat notifications with a pair of LVS directors.

* Use the _ipvsadm_ command to interact directly and/or see what the IPVS kernel component thinks is going on. Keepalived will, in general, do all of this for you, but it doesn't hurt to double check.


~~~~
# ipvsadm
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  red-squid.unl.edu:squid sh
  -> red-squid1.unl.edu:squid     Route   1      0          0
  -> red-squid2.unl.edu:squid     Route   1      0          0
TCP  red-gridftp.unl.edu:gsiftp wlc
  -> red-gridftp6.unl.edu:gsiftp  Route   1      1          14
  -> red-gridftp7.unl.edu:gsiftp  Route   1      1          13
  -> red-gridftp8.unl.edu:gsiftp  Route   1      1          13
  -> red-gridftp10.unl.edu:gsiftp Route   1      1          13
  -> red-gridftp4.unl.edu:gsiftp  Route   1      1          14
  -> red-gridftp5.unl.edu:gsiftp  Route   1      0          15
  -> red-gridftp2.unl.edu:gsiftp  Route   1      1          13
  -> red-gridftp3.unl.edu:gsiftp  Route   1      1          14
  -> red-gridftp9.unl.edu:gsiftp  Route   1      0          13
  -> red-gridftp12.unl.edu:gsiftp Route   1      0          14
TCP  [2600:900:6:1101::da7a:1055]:gsiftp wlc
  -> [2600:900:6:1101:20f:53ff:fe0d:6de0]:gsiftp Route   1      0          0
  -> [2600:900:6:1101:20f:53ff:fe0d:6de4]:gsiftp Route   1      0          0
  -> [2600:900:6:1101:20f:53ff:fe27:3510]:gsiftp Route   1      0          0
  -> [2600:900:6:1101:20f:53ff:fe27:3584]:gsiftp Route   1      0          0
  -> [2600:900:6:1101:20f:53ff:fe27:3588]:gsiftp Route   1      0          0
  -> [2600:900:6:1101:20f:53ff:fe27:3754]:gsiftp Route   1      0          0
  -> [2600:900:6:1101:20f:53ff:fe31:d6a0]:gsiftp Route   1      0          0
  -> [2600:900:6:1101:20f:53ff:fe36:3f90]:gsiftp Route   1      0          0
  -> [2600:900:6:1101:20f:53ff:fe36:3f9c]:gsiftp Route   1      0          0
  -> [2600:900:6:1101:20f:53ff:fe36:3fcc]:gsiftp Route   1      0          0
~~~~


#### Dump IPVS to stdout
Note that the lines are valid ipvsadm commands which you could type yourself. You can also restore this output from stdin with _ipvsadm -R_.
~~~~
# ipvsadm -S
-A -t red-squid.unl.edu:squid -s sh
-a -t red-squid.unl.edu:squid -r red-squid1.unl.edu:squid -g -w 1
-a -t red-squid.unl.edu:squid -r red-squid2.unl.edu:squid -g -w 1
-A -t red-gridftp.unl.edu:gsiftp -s wlc
-a -t red-gridftp.unl.edu:gsiftp -r red-gridftp6.unl.edu:gsiftp -g -w 1
-a -t red-gridftp.unl.edu:gsiftp -r red-gridftp7.unl.edu:gsiftp -g -w 1
-a -t red-gridftp.unl.edu:gsiftp -r red-gridftp8.unl.edu:gsiftp -g -w 1
-a -t red-gridftp.unl.edu:gsiftp -r red-gridftp10.unl.edu:gsiftp -g -w 1
-a -t red-gridftp.unl.edu:gsiftp -r red-gridftp4.unl.edu:gsiftp -g -w 1
-a -t red-gridftp.unl.edu:gsiftp -r red-gridftp5.unl.edu:gsiftp -g -w 1
-a -t red-gridftp.unl.edu:gsiftp -r red-gridftp2.unl.edu:gsiftp -g -w 1
-a -t red-gridftp.unl.edu:gsiftp -r red-gridftp3.unl.edu:gsiftp -g -w 1
-a -t red-gridftp.unl.edu:gsiftp -r red-gridftp9.unl.edu:gsiftp -g -w 1
-a -t red-gridftp.unl.edu:gsiftp -r red-gridftp12.unl.edu:gsiftp -g -w 1
-A -t [2600:900:6:1101::da7a:1055]:gsiftp -s wlc
-a -t [2600:900:6:1101::da7a:1055]:gsiftp -r [2600:900:6:1101:20f:53ff:fe0d:6de0]:gsiftp -g -w 1
-a -t [2600:900:6:1101::da7a:1055]:gsiftp -r [2600:900:6:1101:20f:53ff:fe0d:6de4]:gsiftp -g -w 1
-a -t [2600:900:6:1101::da7a:1055]:gsiftp -r [2600:900:6:1101:20f:53ff:fe27:3510]:gsiftp -g -w 1
-a -t [2600:900:6:1101::da7a:1055]:gsiftp -r [2600:900:6:1101:20f:53ff:fe27:3584]:gsiftp -g -w 1
-a -t [2600:900:6:1101::da7a:1055]:gsiftp -r [2600:900:6:1101:20f:53ff:fe27:3588]:gsiftp -g -w 1
-a -t [2600:900:6:1101::da7a:1055]:gsiftp -r [2600:900:6:1101:20f:53ff:fe27:3754]:gsiftp -g -w 1
-a -t [2600:900:6:1101::da7a:1055]:gsiftp -r [2600:900:6:1101:20f:53ff:fe31:d6a0]:gsiftp -g -w 1
-a -t [2600:900:6:1101::da7a:1055]:gsiftp -r [2600:900:6:1101:20f:53ff:fe36:3f90]:gsiftp -g -w 1
-a -t [2600:900:6:1101::da7a:1055]:gsiftp -r [2600:900:6:1101:20f:53ff:fe36:3f9c]:gsiftp -g -w 1
-a -t [2600:900:6:1101::da7a:1055]:gsiftp -r [2600:900:6:1101:20f:53ff:fe36:3fcc]:gsiftp -g -w 1
~~~~
