# Fortigate Graceful Restart

In a FortiGate High Availability Active/Passive deployment, only the primary unit runs the BGP routing process. When a failover occurs due to a planned event (like maintenance or firmware upgrade) or an unplanned outage (such as hardware failure)—the standby unit becomes the new primary, the new primary must restart the BGP process and reestablish peering sessions. Although the firewall retains its routing table, BGP peers may see the loss of session as a failure and withdraw routes, causing traffic interruption.

The Graceful Restart feature helps traffic disruption. It allows BGP peers to keep routes temporarily during the transition, treating the session loss as a temporary event. If the FortiGate reestablishes the BGP session within the restart timer (typically 120 seconds), routes are refreshed without disruption.
This ensures routing stability, prevents unnecessary route flapping, and minimises downtime during HA failover events.

The output below confirms the disruption when failover from the primary to secondary unit occurs and graceful restart is not enabled.

```ruby
Switch#ping 8.8.8.8 repeat 10000
Type escape sequence to abort.
Sending 10000, 100-byte ICMP Echos to 8.8.8.8, timeout is 2 seconds:
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!!!!!!!!!!!!!!.!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!U.U.U.U.U.U.U.U.U.U.U.U.U.U
.U.U.U.U.U.U.U.U.U.U.U.U.U.U.U.U.U.U.U.U.U.U.U.U.U.U.U.U.U.U.U.U.U.U.U
.U.U.U.U.U.U.U.U.U.U.U.U.U.U.U.U.U.U.U.U.U.U.U.U.U.U.U.U.U.U.!!!!!!!!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!.
Success rate is 78 percent (597/757), round-trip min/avg/max = 12/17/64 ms
```
From the logs on the Cisco switch the FortiGate is connected to, we can determine that BGP took over 120 seconds to re-establish a BGP adjacency to the new active Fortigate Firewall.

```ruby
*Jul  5 15:54:07.789: %BGP-5-NBR_RESET: Neighbor 1.1.1.2 passive reset (BGP Notification sent)
*Jul  5 15:54:07.809: %BGP-5-ADJCHANGE: neighbor 1.1.1.2 passive Down Error during connection collision
*Jul  5 15:54:09.332: %BGP-3-NOTIFICATION: sent to neighbor 1.1.1.2 passive 6/7 (Connection Collision Resolution) 0 bytes
*Jul  5 15:54:13.962: %BGP-5-NBR_RESET: Neighbor 1.1.1.2 passive reset (BGP Notification sent)
*Jul  5 15:54:13.973: %BGP-5-ADJCHANGE: neighbor 1.1.1.2 passive Down Error during connection collision
*Jul  5 15:54:14.313: %BGP-3-NOTIFICATION: sent to neighbor 1.1.1.2 passive 6/7 (Connection Collision Resolution) 0 bytes
*Jul  5 15:54:19.083: %BGP-5-NBR_RESET: Neighbor 1.1.1.2 passive reset (BGP Notification sent)
*Jul  5 15:54:19.094: %BGP-5-ADJCHANGE: neighbor 1.1.1.2 passive Down Error during connection collision
*Jul  5 15:54:19.251: %BGP-3-NOTIFICATION: sent to neighbor 1.1.1.2 passive 6/7 (Connection Collision Resolution) 0 bytes
*Jul  5 15:54:24.204: %BGP-5-NBR_RESET: Neighbor 1.1.1.2 passive reset (BGP Notification sent)
*Jul  5 15:54:24.218: %BGP-5-ADJCHANGE: neighbor 1.1.1.2 passive Down Error during connection collision
*Jul  5 15:54:25.246: %BGP-3-NOTIFICATION: sent to neighbor 1.1.1.2 passive 6/7 (Connection Collision Resolution) 0 bytes
*Jul  5 15:54:30.409: %BGP-5-NBR_RESET: Neighbor 1.1.1.2 passive reset (BGP Notification sent)
*Jul  5 15:54:30.426: %BGP-5-ADJCHANGE: neighbor 1.1.1.2 passive Down Error during connection collision
*Jul  5 15:56:27.269: %BGP-3-NOTIFICATION: sent to neighbor 1.1.1.2 passive 6/7 (Connection Collision Resolution) 0 bytes
*Jul  5 15:56:32.278: %BGP-5-NBR_RESET: Neighbor 1.1.1.2 passive reset (BGP Notification sent)
*Jul  5 15:56:32.286: %BGP-5-ADJCHANGE: neighbor 1.1.1.2 passive Down Error during connection collision
*Jul  5 15:56:35.792: %BGP-3-NOTIFICATION: sent to neighbor 1.1.1.2 4/0 (hold time expired) 0 bytes
*Jul  5 15:56:35.796: %BGP-5-NBR_RESET: Neighbor 1.1.1.2 reset (BGP Notification sent)
*Jul  5 15:56:35.804: %BGP-5-ADJCHANGE: neighbor 1.1.1.2 Down BGP Notification sent
*Jul  5 15:56:35.805: %BGP_SESSION-5-ADJCHANGE: neighbor 1.1.1.2 IPv4 Unicast topology base removed from session  BGP Notification sent
*Jul  5 15:56:44.646: %BGP-5-ADJCHANGE: neighbor 1.1.1.2 Up
```

# Configuration
On a Cisco switch/router, graceful-restart is configured globally under BGP. The CLI syntax below represents the entire configuration used in this scenario.
```ruby
router bgp 65000
 bgp router-id 1.1.1.254
 bgp log-neighbor-changes
 bgp graceful-restart
 neighbor 1.1.1.2 remote-as 65001
 neighbor 1.1.1.2 version 4
 !
 address-family ipv4
  redistribute connected
  neighbor 1.1.1.2 activate
  neighbor 1.1.1.2 default-originate
 exit-address-family
```
Graceful restart on the FortiGate firewall is configured globally in the BGP settings and must also be enabled individually for each BGP peer.
Navigate to Network > BGP > Graceful Restart and select Graceful Restart
 
Navigate to Network > BGP > Neighbors
Edit the specific neighbor and tick Capability:graceful restart

Graceful restart can also be configured using CLI, the output below represents the entire BGP configuration to establish adjacency to a peer and configure graceful-restart globally and for the specific peer.

```ruby
config router bgp
    set as 65001
    set router-id 1.1.1.2
    set graceful-restart enable
    config neighbor
        edit "1.1.1.254"
            set capability-graceful-restart enable
            set remote-as 65000
        next
    end
    config network
        edit 1
            set prefix 1.1.1.0 255.255.255.0
        next
    end
    config network6
        edit 1
            set prefix6 ::/128
        next
    end
    config redistribute "connected"
    end
    config redistribute "rip"
    end
    config redistribute "ospf"
    end
    config redistribute "static"
    end
    config redistribute "isis"
    end
    config redistribute6 "connected"
    end
    config redistribute6 "rip"
    end
    config redistribute6 "ospf"
    end
    config redistribute6 "static"
    end
    config redistribute6 "isis"
    end
end
```

# Verification
With graceful restart enabled on both FortiGate firewalls and the connected Cisco switches, we can force a failover of the HA pair and observe the outcome.
Run a ping that is routed through the FortiGate firewall and observe the output.
Enable BGP debugs on the FortiGate Firewalls (both active and secondary)

**Enable BGP debugs: **
 
diagnose ip router bgp all enable  
diagnose ip router bgp level info  
diagnose debug enable  
 
To disable BGP debugs:  
 
diagnose ip router bgp all disable  
diagnose ip router bgp level none  
diagnose debug reset  

From the active FortiGate we initiate a failover to the secondary device using the command execute ha failover set 1

```ruby
Caution: This command will trigger an HA failover. 
It is intended for testing purposes.
Do you want to continue? (y/n)y
HQ-FGT1 # execute ha failover set 1
Caution: This command will trigger an HA failover. 
It is intended for testing purposes.
Do you want to continue? (y/n)y

The output confirms a brief blip which coincides with failing over to the secondary FortiGate and only 1 packet lost.
Switch#ping 8.8.8.8 repeat 10000
Type escape sequence to abort.
Sending 10000, 100-byte ICMP Echos to 8.8.8.8, timeout is 2 seconds:
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!!!!!.!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
Success rate is 99 percent (617/618), round-trip min/avg/max = 12/17/54 ms
Switch#
```

When failover is initiated on the current active FortiGate Firewall, we see the BGP routes withdrawn and BGP adjacency is Down and peer being deleted.

```ruby
BGP: 1.1.1.254-Outgoing [FSM] State: Established Event: 2
BGP: VRF 0 NSM withdraw: 2.2.2.0/24
BGP: VRF 0 NSM withdraw: 3.3.3.0/24
BGP: VRF 0 NSM withdraw: 192.168.21.0/24
BGP: VRF 0 NSM withdraw: 0.0.0.0/0
id=20300 msg="BGP: %BGP-5-ADJCHANGE: VRF 0 neighbor 1.1.1.254 Down Peer being deleted"
```
On the now active FortiGate Firewall we can see BGP Up almost at the same time failover occurred.
```ruby
BGP: 1.1.1.254-Outgoing [DECODE] Open Cap: unrecognized capability code 70 len 0
id=20300 msg="BGP: %BGP-5-ADJCHANGE: VRF 0 neighbor 1.1.1.254 Up "
```
The logs on the Cisco switch provide more accurate information and confirms when the BGP adjacency went down and come up again.
```ruby
*Jul  5 16:09:39.493: %BGP_SESSION-5-ADJCHANGE: neighbor 1.1.1.2 IPv4 Unicast topology base removed from session  NSF peer closed the session
*Jul  5 16:09:39.499: %BGP-5-NBR_RESET: Neighbor 1.1.1.2 reset (NSF peer closed the session)
*Jul  5 16:09:39.505: %BGP-5-ADJCHANGE: neighbor 1.1.1.2 Down NSF peer closed the session
*Jul  5 16:09:39.526: %BGP-5-ADJCHANGE: neighbor 1.1.1.2 Up
```
To check the FortiGate graceful restart is enable run the command get router bgp
```ruby
HQ-FGT1 # get router bgp 
as                  : 65001
router-id           : 1.1.1.2
keepalive-timer     : 60
holdtime-timer      : 180
always-compare-med  : disable 
bestpath-as-path-ignore: disable 
bestpath-cmp-confed-aspath: disable 
bestpath-cmp-routerid: disable 
bestpath-med-confed : disable 
bestpath-med-missing-as-worst: disable 
client-to-client-reflection: enable 
dampening           : disable 
deterministic-med   : disable 
ebgp-multipath      : disable 
ibgp-multipath      : disable 
enforce-first-as    : enable 
fast-external-failover: enable 
log-neighbour-changes: enable 
network-import-check: enable 
ignore-optional-capability: enable 
multipath-recursive-distance: disable 
recursive-next-hop  : disable 
recursive-inherit-priority: disable 
tag-resolve-mode    : disable 
cluster-id          : 0.0.0.0
confederation-identifier: 0
default-local-preference: 100
scan-time           : 60
distance-external   : 20
distance-internal   : 200
distance-local      : 200
synchronization     : disable 
graceful-restart    : enable 
graceful-end-on-timer: disable 
cross-family-conditional-adv: disable 
aggregate-address:
aggregate-address6:
neighbor:
    == [ 1.1.1.254 ]
    ip:     1.1.1.254       auth-options:        
neighbor-group:
neighbor-range:
neighbor-range6:
network:
    == [ 1 ]
    id:     1   
network6:
    == [ 1 ]
    id:     1   
redistribute:
    == [ connected ]
    name:     connected       status: disable            route-map:        
    == [ rip ]
    name:     rip       status: disable            route-map:        
    == [ ospf ]
    name:     ospf       status: disable            route-map:        
    == [ static ]
    name:     static       status: disable            route-map:        
    == [ isis ]
    name:     isis       status: disable            route-map:        
redistribute6:
    == [ connected ]
    name:     connected       status: disable            route-map:        
    == [ rip ]
    name:     rip       status: disable            route-map:        
    == [ ospf ]
    name:     ospf       status: disable            route-map:        
    == [ static ]
    name:     static       status: disable            route-map:        
    == [ isis ]
    name:     isis       status: disable            route-map:        
admin-distance:
vrf:
vrf6:
graceful-restart-time: 120
graceful-stalepath-time: 360
graceful-update-delay: 120
```
# Summary
As observed in this post, we can see the benefits of using graceful restart between a FortiGate HA pair and the connected Cisco switches to achieve fast failover and avoid traffic disruption. Graceful restart can also be configured with other routing protocols i.e, OSPF, IS-IS etc, not just BGP.
