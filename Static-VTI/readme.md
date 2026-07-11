# VTI Configuration
Configure the IKE (Phase 1) settings, this includes the IKE version (1 or 2), key lifetime, IKE proposal, DH group, authentication method (pre-shared key or certificates), Pre-Shared-Key, interface and remote gateway IP address.
```ruby
config vpn ipsec phase1-interface
    edit "to-SPOKE1"
        set interface "port1"
        set ike-version 2
        set keylife 43200
        set peertype any
        set net-device enable
        set proposal aes256gcm-prfsha512
        set dhgrp 21
        set remote-gw 1.0.1.1
        set psksecret ENC 9ibbjWh0GLx7AH/UAOxT2RpzDJYcX/3Of6J1vshOpRQOR5f6sibjWY9lnT42pTi+CZ3RU7jzBaJWUqSzqqNywI9id8R5lrp+0I4Er12KP8YqobijqjkTxS/zJ+3FGKBcsQ92MkRA4Tg5JTJ9uxcN4wCWUr/rxuYpAOBV1GUVIBV326hMTTR68BSQeamWvZJ/dk1heA==
    next
end
'''

The IPsec (Phase 2) configuration references the already created Phase 1 settings and specifies the IPSec proposal, DH group (if using PFS) and key lifetime of Phase 2.
config vpn ipsec phase2-interface
    edit "to-SPOKE1"
        set phase1name "to-SPOKE1"
        set proposal aes256gcm
        set dhgrp 21
        set keylifeseconds 3600
    next
end
Configuring the Phase 1 interface will automatically create a logical interface with the same name and the basic settings.

DC1-HUB # show | grep -f to-SPOKE1
config system interface
    edit "to-SPOKE1"
        set vdom "root"
        set type tunnel
        set snmp-index 9
        set interface "port1"
    next
end

Edit the tunnel interface and configure the tunnel IP address, this must be configured as a /32 and specify the “remote-ip” of the peer’s tunnel IP address.

config system interface
    edit "to-SPOKE1"
        set vdom "root"
        set ip 10.5.0.1 255.255.255.255
        set allowaccess ping
        set type tunnel
        set remote-ip 10.5.0.2 255.255.255.252
        set snmp-index 9
        set interface "port1"
    next
end

NOTE – if you do not specify the “remote-ip” of the peer’s tunnel IP address, there will be no dynamically created route to the peer’s tunnel IP address in the routing table and the FortiGate will not automatically be able to communicate with the peer’s tunnel IP address.

Edit the firewall policy and create rules to communicate over the VPN (inbound and outbound).

config firewall policy
    edit 3
        set name "to SPOKE1"
        set srcintf "port3"
        set dstintf "to-SPOKE1"
        set action accept
        set srcaddr "all"
        set dstaddr "all"
        set schedule "always"
        set service "ALL"
        set logtraffic all
    next
    edit 4
        set name "from SPOKE1"
        set srcintf "to-SPOKE1"
        set dstintf "port3"
        set action accept
        set srcaddr "all"
        set dstaddr "all"
        set schedule "always"
        set service "ALL"
        set logtraffic all
     next
end

Verification
At this point the tunnel should automatically be established if the peer device is already configured and configured correctly.
Run the command diagnose vpn ike gateway, this will show the IKE (Phase 1) gateways and state. You can determine if the tunnel is up, look for status: established.
DC1-HUB # diagnose vpn ike gateway
vd: root/0
name: to-SPOKE1
version: 2
interface: port1 3
addr: 1.0.0.1:500 -> 1.0.1.1:500
tun_id: 1.0.1.1/::1.0.1.1
remote_location: 0.0.0.0
network-id: 0
transport: UDP
virtual-interface-addr: 10.5.0.1 -> 10.5.0.2
created: 110s ago
peer-id: 1.0.1.1
peer-id-auth: no
PPK: no
IKE SA: created 1/2  established 1/2  time 20/45/70 ms
IPsec SA: created 1/2  established 1/2  time 0/35/70 ms

  id/spi: 47 45f2c1131891d3ce/5132fa7ea1ca31db
  direction: initiator
  status: established 110-110s ago = 70ms
  proposal: aes256gcm
  child: no
  SK_ei: 54d36d758b3e55bb-23a9d1e80c4f303c-8e259ca953a8c6ab-35564528aedd0477-ecd02450
  SK_er: 85df83adb26f1a97-ef1bc91b35d05d1e-b5f1412112a5d913-7f849a94ff3525c1-45dcbf1f
  SK_ai: 
  SK_ar: 
  PPK: no
  message-id sent/recv: 2/3
  QKD: no
  lifetime/rekey: 43200/42789
  DPD sent/recv: 00000000/00000000
  peer-id: 1.0.1.1

Run the command diagnose vpn tunnel list. Show the state of the IPSec tunnels (Phase 2/ CHILD SA) – this will show the lifetime, encrypted/decrypted packets amongst other settings.

DC1-HUB # diagnose vpn tunnel list 
list all ipsec tunnel in vd 0
------------------------------------------------------
name=to-SPOKE1 ver=2 serial=2 1.0.0.1:0->1.0.1.1:0 tun_id=1.0.1.1 tun_id6=::1.0.1.1 dst_mtu=1500 dpd-link=on weight=1
bound_if=3 lgwy=static/1 tun=intf mode=auto/1 encap=none/568 options[0238]=npu create_dev frag-rfc  role=primary accept_traffic=1 overlay_id=0

proxyid_num=1 child_num=0 refcnt=4 ilast=6 olast=106 ad=/0
stat: rxp=17 txp=2 rxb=1128 txb=168
dpd: mode=on-demand on=1 idle=20000ms retry=3 count=0 seqno=0
natt: mode=none draft=0 interval=0 remote_port=0
fec: egress=0 ingress=0
proxyid=to-SPOKE1 proto=0 sa=1 ref=2 serial=1
  src: 0:0.0.0.0-255.255.255.255:0
  dst: 0:0.0.0.0-255.255.255.255:0
  SA:  ref=3 options=30202 type=00 soft=0 mtu=1438 expire=3148/0B replaywin=2048
       seqno=3 esn=0 replaywin_lastseq=00000012 qat=0 rekey=0 hash_search_len=1
  life: type=01 bytes=0/0 timeout=3300/3600
  dec: spi=200e9861 esp=aes-gcm key=36 4584199e4d701054103bbfec0fddb2c4d5c0a9d33dcabc010eb3b673c31a196e63dca186
       ah=null key=0 
  enc: spi=c16ed423 esp=aes-gcm key=36 f5b0b582a381dd366a424c5ebc47dce741db89b2c2b706818e76f069ee72742e0f11043b
       ah=null key=0 
  dec:pkts/bytes=1237/1128, enc:pkts/bytes=2/296
  npu_flag=00 npu_rgwy=1.0.1.1 npu_lgwy=1.0.0.1 npu_selid=1 dec_npuid=0 enc_npuid=0

With Phase 1 and Phase 2 successfully established, confirm the route to the tunnel network has automatically been added to the routing table, run the command get router info routing-table all.
From the output below we can determine there is the route to the tunnel network via the tunnel interface “to-SPOKE1”.
 
Test communication to the peer’s tunnel IP address execute ping <peer’s tunnel ip>.
 
We can confirm the VPN status from the GUI, navigate to the IPSec Monitor, this will confirm the tunnel is up and the incoming/outgoing data in bytes.
 
Useful Commands
The following are useful commands when troubleshooting FortiGate VPN issues: -
To flush the tunnel:
diagnose vpn tunnel flush <my-phase1-name>
diagnose vpn ike gateway clear name <my-phase1-name>
diagnose vpn ike gateway flush name <my-phase1-name>

Debug commands:
diagnose vpn ike log filter rem-addr4 <peer public ip address>
diagnose debug application ike -1
diagnose debug application fnbamd -1
diagnose debug application eap_proxy -1
diagnose debug console timestamp enable
diagnose debug enable
 
Disable debugs:
diagnose debug reset
diagnose debug disable

Resources
https://community.fortinet.com/t5/FortiGate/Troubleshooting-Tip-IPsec-VPN-tunnels/ta-p/195955
https://community.fortinet.com/t5/FortiGate/Troubleshooting-Tip-Troubleshooting-IPsec-Site-to-Site-Tunnel/ta-p/195672
