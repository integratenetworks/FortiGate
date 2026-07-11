# FortiGate BGP Community List

BGP communities are optional tags that can be attached to routes and carried between routers. They let you tag routes at ingress and match on those tags in your outbound policy, controlling what gets advertised to which peers without maintaining individual prefix configurations per neighbour. This makes them particularly useful for controlling route leaking between peers at scale.
Background

The diagram below represents the topology covered in post.
 
In this scenario the FortiGate Firewall (DCFW) has eBGP relationships with PEER1 and PEER2, all devices have each other’s routes in their routing table.  This FortiGate will set a community value to all routes learned from PEER1 and only allow routes matching that community value to be advertised to PEER2.
Run the command get router info routing-table bgp on PEER2 to confirm it has DCFW and PEER1 routes.
```ruby
PEER2 # get router info routing-table bgp
Routing table for VRF=0
B       172.16.0.0/24 [20/0] via 192.168.250.1 (recursive is directly connected, port11), 00:43:19, [1/0]
B       172.16.1.0/24 [20/0] via 192.168.250.1 (recursive is directly connected, port11), 00:43:19, [1/0]
B       172.16.2.0/24 [20/0] via 192.168.250.1 (recursive is directly connected, port11), 00:43:19, [1/0]
B       192.168.101.0/24 [20/0] via 192.168.250.1 (recursive is directly connected, port11), 00:52:15, [1/0]
B       192.168.102.0/24 [20/0] via 192.168.250.1 (recursive is directly connected, port11), 00:47:00, [1/0]
```
Run the command get router info bgp network <network> and observe the output, there will be no “Community: <value>” in the output of a PEER1 network.
```ruby
PEER2 # get router info bgp network 172.16.0.0/24
VRF 0 BGP routing table entry for 172.16.0.0/24
Paths: (1 available, best #1, table Default-IP-Routing-Table)
  Not advertised to any peer
  Original VRF 0
  65000 65001
    192.168.250.1 from 192.168.250.1 (172.21.1.1)
      Origin IGP distance 20 metric 0, localpref 100, valid, external, best
      Last update: Thu Mar 26 15:13:12 2026
```
# Configuration
Basic BGP configuration is applied to the FortiGate, establishing BGP adjacencies to PEER1 (192.168.251.2) and PEER2 (192.168.250.2), with no inbound or outbound filtering. The FortiGate advertises its own networks (192.168.101.0/24 and 192.168.102.0/24).
```ruby
config router bgp
    set as 65000
    set router-id 172.21.1.1
    set graceful-restart enable
    config neighbor
        edit "192.168.251.2"
            set soft-reconfiguration enable
            set description "PEER1"
            set remote-as 65001
            set keep-alive-timer 2
            set holdtime-timer 6
        next
        edit "192.168.250.2"
            set soft-reconfiguration enable
            set description "PEER2"
            set remote-as 65002
            set keep-alive-timer 2
            set holdtime-timer 6
        next
    end
    config network
        edit 1
            set prefix 192.168.101.0 255.255.255.0
        next
        edit 2
            set prefix 192.168.102.0 255.255.255.0
        next
    end
```
A route-map is created to be applied inbound from PEER1 that sets the community value.
```ruby
config router route-map
    edit "PEER1-IN"
        config rule
            edit 1
                set set-community "65001:1"
            next
        end
    next
```
To match on the community outbound, a community-list is created which matches on the value specified in the PEER1-IN route-map.
```ruby
config router community-list
    edit "MATCH-COMMUNITY"
        config rule
            edit 1
                set action permit
                set match "65001:1"
            next
        end
    next
end
```
A second route-map is configured to match routes only with the set community, denying all other routes. This will be applied outbound towards PEER2, filtering the routes accordingly.
```ruby
config router route-map
    edit "PEER2-OUT"
        config rule
            edit 1
                set match-community "MATCH-COMMUNITY"
            next
            edit 2
                set action deny
            next
        end
    next
end
```
The PEER1-IN route-map must be configured on PEER1 neighbour, inbound.
```ruby
config router bgp
    config neighbor
        edit "192.168.251.2"
            set soft-reconfiguration enable
            set description "PEER1"
            set remote-as 65001
            set route-map-in "PEER1-IN"
            set keep-alive-timer 2
            set holdtime-timer 6
        next
```
The PEER2-OUT route-map must be configured on PEER2 neighbour, outbound.
```ruby
        edit "192.168.250.2"
            set soft-reconfiguration enable
            set description "PEER2"
            set remote-as 65002
            set route-map-out “PEER2-OUT”
            set keep-alive-timer 2
            set holdtime-timer 6
        next
    end
```
After the route-map PEER1-IN has been applied on DCFW wait or refresh the routes using the command execute router clear bgp all soft. 

# Verification
With the route-maps configured on the FortiGate Firewall DCFW, run the command get router info bgp network 172.16.0.0/24 to confirm it has applied the community to routes learned from PEER1.
```ruby
DCFW # get router info bgp network 172.16.0.0/24
VRF 0 BGP routing table entry for 172.16.0.0/24
Paths: (1 available, best #1, table Default-IP-Routing-Table)
  Advertised to non peer-group peers:
   192.168.250.2
  Original VRF 0
  65001
    192.168.251.2 from 192.168.251.2 (192.168.0.1)
      Origin IGP distance 20 metric 0, localpref 100, valid, external, best
      Community: 65001:1
      Last update: Thu Mar 26 08:16:51 2026
```
Connecting to PEER2 and observe the routing table, run the command get router info routing-table bgp on PEER2 to confirm it now only has the PEER1 routes (172.16.0.0/24, 172.16.1.0/24 and 172.16.2.0/24), the DCFW routes are not present as they were not assigned the community value, so were denied advertising from the DCFW.
```ruby
PEER2 # get router info routing-table bgp
Routing table for VRF=0
B       172.16.0.0/24 [20/0] via 192.168.250.1 (recursive is directly connected, port11), 00:00:06, [1/0]
B       172.16.1.0/24 [20/0] via 192.168.250.1 (recursive is directly connected, port11), 00:00:06, [1/0]
B       172.16.2.0/24 [20/0] via 192.168.250.1 (recursive is directly connected, port11), 00:00:06, [1/0]
```
Run the command get router info bgp network <network> and confirm the Community is set on the route.
```ruby
PEER2 # get router info bgp network 172.16.0.0/24
VRF 0 BGP routing table entry for 172.16.0.0/24
Paths: (1 available, best #1, table Default-IP-Routing-Table)
  Not advertised to any peer
  Original VRF 0
  65000 65001
    192.168.250.1 from 192.168.250.1 (172.21.1.1)
      Origin IGP distance 20 metric 0, localpref 100, valid, external, best
      Community: 65001:1
      Last update: Thu Mar 26 16:38:36 2026
```
# Summary
In this post a FortiGate Firewall was configured to assign a BGP community value to all routes learned from PEER1 using an inbound route-map. A community-list was then used to match on that value in an outbound route-map applied towards PEER2, ensuring only routes tagged with the community were advertised. The result was confirmed on PEER2, which went from receiving all routes to receiving only PEER1 routes, with the community value visible in the BGP table. This demonstrates how communities can be used as a scalable and clean mechanism for controlling route advertisement between peers without the need for prefix-list based filtering.
