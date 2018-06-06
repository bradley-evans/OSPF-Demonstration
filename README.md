# Open Shortest Path First

Final Project Demonstration for CS183, UNIX Systems Administration (Spring 2018), at the University of California, Riverside.

Prepared by Bradley Evans

## Introduction ##

In a static routing system, we must explicitly define our routes from one network to the other via a routing table. We have to establish default routes, routes that are associated with specific destinations, and so on. This becomes untenable as the network grows in size and complexity. Moreover, such a statically-defined network is not responsive to change -- a node disappearing from the network may result in loss of connectivity to or from downstream services, despite the presence of redundant paths.

Open Shortest Path First (OSPF) is a dynamic routing protocol. 

### General Schema ###

OSPF routers form self-healing mesh networks with each other. This is accomplished using a mechanism called an _adjacency state machine,_ where each node in the network is made aware of its neighboring nodes using special messaging packets. The network elects a _designated router_ and a _backup designated router_. These routers contain a complete topology table of the network, and aid in determining the shortest path between two points in the network.

The topology is established by routers sending special, OSPF-specific packets. The most important of these is the _hello_ packet, which allows OSPF-enabled routers to discover other adjacent devices and build the topology table. Other packets include _database description (DBD)_ messages, _link state requests_, _link state updates_, and _link state acknowledgements_ which together allow the network to observe changes to the topology.

The actual shortest path between two points is calculated using Djikstra's algorithm.

### Areas ###

OSPF networks may be divided into areas. In our implementation below, the three routers R1-R3 form _Area 0_, or the backbone area. This is the area to which all other areas are connected, or the "core" of the network. We define two other areas in our implementation below, namely Area 1 (`10.0.10.0/24`) and Area 2 (`10.0.11.0/24`).

## Implementation ##

### Tools and Software Utilized ###

Our implentation of OSPF was performed using the Graphical Network Simulator (GNS3, https://www.gns3.com/).

The simulation was performed on a Thinkpad Yoga P40 running Windows 10. Virtual machines were supported on the backend by VMWare Workstation Professional 14.1.1. Other than VMWare Workstation (which is proprietary but interchangeable with another virtualization solution like Virtualbox), the software used was free. Images used for the routers and boxes were legally obtained with GNS3.

### Network Topology ###

In GNS3, we define the topology as so.

![](https://i.imgur.com/YWXTWKt.png)

This is not the most complex implementation of OSPF possible, but it is sufficient to demonstrate the protocol and learn how to set it up.

The routers R1, R2, and R3 are Cisco 3745 router images provided with GNS3. These images were selected more or less arbitrarily from a list of available equipment that supports OSPF. 

The switches SW1 and SW2 are generic Layer 2 switches provided with GNS3.

The endpoints PC1 and PC2 are generic "Virtual PC Simulators" that only act to receive basic pings and demonstrate connectivity.

### Router Configuration ###

To configure the Cisco 3745 routers, we open a terminal to each by double clicking on each router in GNS3. For brevity, only the configuration for Router 1 will be explained in detail, but the steps are rapidly modifiable for other routers in the topology.

For *Router 1*,

1. Enter configuration mode. `conf t`
2. Set up the interfaces.
	1. `interface fastethernet 0/1` gets us into configuration mode for that specific interface.
	2. `ip address 10.0.10.1 255.255.255.0` to define the interface's IP address.
	3. `no shut` to bring the interface `up` 
	4. `end` to save the configuration for that interface.
	5. Repeat for every other interface. Assign `10.0.32.2/24` to `FastEthernet1/0`, and `10.0.12.2/24` to `FastEthernet0/0`.
3. Set up OSPF.
	1. From configuration mode, `router ospf 1` will allow us to set up an OSPF profile for the router.
	2. `network 10.0.10.0 0.0.0.255 area 2` places the PC1 endpoint in Area 2 of the OSPF topology and defines it as an OSPF interface.
	3. `network 10.0.23.0 0.0.0.255 area 1` defines the network on interface 1/0, and `network 10.0.12.0 0.0.0.255 area 1` does the same on 0/0. Note the reverse masks here.
	4. `end` to save changes.
5. `write` from outside of configuration mode will save the running configuration to the startup configuration.

Once this process is repeated on each router, we have a fully functional network.

### Endpoint Configuration ###

Some final configuration is needed on the endpoints. For each we need to assign them an IP and a gateway. For now, this can be achieved statically. For *PC1*, starting up the machine and entering `ip 10.0.10.100 10.0.10.1` suffices to bring the machine into the network. PC2 is configured in similar fashion.

## Simulation ##

### Inter-Router Communications ###

Bringing all of the nodes online allows us to start examining the packets traversing the network. We can right click on a link and start a capture.

![](https://i.imgur.com/Pq6Gsuo.png)

From within Wireshark, we can see the hello and reply packets traversing the link. An example hello packet looks like this.

```
53	119.998120	10.0.12.2	224.0.0.5	OSPF	94	Hello Packet
```

![](https://i.imgur.com/7iMh1yX.png)

These packets include all of the link state data. It shows the designated router in the topology, the backup designated router, active neighbors, and so on. 

### Endpoint Testing ###

We ensure the endpoints have IP addresses and that they can communicate to each other (that ICMP pings are getting routed at all).

```

PC-1> show ip

NAME        : PC-1[1]
IP/MASK     : 10.0.10.100/24
GATEWAY     : 10.0.10.1
DNS         :
MAC         : 00:50:79:66:68:00
LPORT       : 10002
RHOST:PORT  : 127.0.0.1:10003
MTU:        : 1500

PC-1> ping 10.0.11.100
84 bytes from 10.0.11.100 icmp_seq=1 ttl=62 time=34.523 ms
84 bytes from 10.0.11.100 icmp_seq=2 ttl=62 time=40.949 ms
84 bytes from 10.0.11.100 icmp_seq=3 ttl=62 time=39.142 ms
84 bytes from 10.0.11.100 icmp_seq=4 ttl=62 time=28.662 ms
84 bytes from 10.0.11.100 icmp_seq=5 ttl=62 time=34.936 ms
```

Looks good. A traceroute gives us the following.

```
PC-1> trace 10.0.11.100
trace to 10.0.11.100, 8 hops max, press Ctrl+C to stop
 1   10.0.10.1   6.508 ms  10.713 ms  12.069 ms
 2   10.0.23.1   29.909 ms  22.353 ms  22.926 ms
 3   *10.0.11.100   37.282 ms
```

This traceroute shows a valid "shortest path." We can trace the path through the topology graph above. A good way to test the "self-healing" nature of an OSPF network is to cut a link and observe what happens. We'll remove the R1<->R3 link and observe how this changes the path of the packets through the network.

Once the network converges, we can repeat the trace from one of our endpoints.

```
PC-1> trace 10.0.11.100
trace to 10.0.11.100, 8 hops max, press Ctrl+C to stop
 1   10.0.10.1   7.293 ms  9.181 ms  9.555 ms
 2   10.0.12.1   53.212 ms  49.093 ms  29.243 ms
 3   10.0.13.2   48.316 ms  40.404 ms  40.675 ms
 4   *10.0.11.100   47.314 ms
```

The trace shows an added hop and a new path through R2. The network has re-converged and overcome a lost link. This demonstrates OSPF's value in adding resiliency to a network.

## Conclusions ##

OSPF has clear advantages over static routing methods. The network becomes more resilient to changes, especially unplanned changes (outages), assuming the administrator plans redundant routes into the topology. The network is largely self-configuring and highly scalable. Such a system could easily grow to encompass, say, a business campus with new areas being added to the topology on an ad-hoc basis.

We were able to demonstrate such a network above using free tools on simulated Cisco devices. The above demonstration allowed us to see packets in-flight, and we were even able to understand the lag-time between network changes and the time to convergence on the new topology table. 