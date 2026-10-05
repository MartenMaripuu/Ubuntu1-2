
Addressing plan

VM1 (ubuntu4), enp0s9, labnet-a, 10.10.10.20/24, fd00:10::20/64
VM2 (ubuntu6), enp0s9, labnet-b, 10.10.20.20/24, fd00:20::20/64
Router (ubuntu8), enp0s9, labnet-a, 10.10.10.1/24, fd00:10::1/64
Router (ubuntu8), enp0s10, labnet-b, 10.10.20.1/24, fd00:20::1/64

VM1 also has enp0s3 on DHCP (bridged, used for SSH) and enp0s8 (host-only, marked optional in Netplan due to a host driver issue).
Router also has enp0s3 on DHCP (bridged, used for SSH). Host-only adapter was not used due to the same driver issue.
Rocky Linux was skipped for Lab 5.A; VM2 (ubuntu6, Ubuntu with nmcli) was used as the LAN B host in its place.

VM1 Netplan config, final version with routes and DNS

network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      addresses: [192.168.56.10/24]
      optional: true
    enp0s9:
      addresses: [10.10.10.20/24, "fd00:10::20/64"]
      routes:
        - to: 10.10.20.0/24
          via: 10.10.10.1
        - to: fd00:20::/64
          via: fd00:10::1
      nameservers:
        addresses: [10.10.10.1]

VM2 nmcli config

ipv4.method:    manual
ipv4.addresses: 10.10.20.20/24
ipv6.method:    manual
ipv6.addresses: fd00:20::20/64

Routes added via:
sudo nmcli con mod lanb +ipv4.routes "10.10.10.0/24 10.10.20.1"
sudo nmcli con mod lanb +ipv6.routes "fd00:10::/64 fd00:20::1"
sudo nmcli con up lanb

DNS added via:
sudo nmcli con mod lanb ipv4.dns "10.10.20.1"
sudo nmcli con up lanb

Router Netplan config

network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s9:
      addresses: [10.10.10.1/24, "fd00:10::1/64"]
    enp0s10:
      addresses: [10.10.20.1/24, "fd00:20::1/64"]

Verification, ip addr and ip link, all three VMs

VM1:
2: enp0s3: mtu 1500 state UP
    link/ether 08:00:27:6f:6a:cc
    inet 172.18.25.42/24
3: enp0s9: mtu 1500 state UP
    link/ether 08:00:27:ab:72:a4
    inet 10.10.10.20/24
    inet6 fd00:10::20/64

VM2:
2: enp0s3: mtu 1500 state UP
    link/ether 08:00:27:2a:2f:52
    inet 172.18.25.35/24
3: enp0s9: mtu 1500 state UP
    link/ether 08:00:27:90:90:99
    inet 10.10.20.20/24
    inet6 fd00:20::20/64

Router:
2: enp0s3: mtu 1500 state UP
    link/ether 08:00:27:5f:5d:d9
    inet 172.18.25.44/24
3: enp0s9: mtu 1500 state UP
    link/ether 08:00:27:af:c3:75
    inet 10.10.10.1/24
    inet6 fd00:10::1/64
4: enp0s10: mtu 1500 state UP
    link/ether 08:00:27:df:eb:ff
    inet 10.10.20.1/24
    inet6 fd00:20::1/64

Isolation test, before the router existed

Self-ping, success:
PING 10.10.10.20 (10.10.10.20) 56(84) bytes of data.
64 bytes from 10.10.10.20: icmp_seq=1 ttl=64 time=0.028 ms
64 bytes from 10.10.10.20: icmp_seq=2 ttl=64 time=0.064 ms
64 bytes from 10.10.10.20: icmp_seq=3 ttl=64 time=0.055 ms
3 packets transmitted, 3 received, 0% packet loss

Cross-VM ping, fails, no router yet:
PING 10.10.20.20 (10.10.20.20) 56(84) bytes of data.
3 packets transmitted, 0 received, 100% packet loss, time 2076ms

Explanation: labnet-a and labnet-b are separate internal networks in VirtualBox, isolated Layer 2 segments with no router or switch connecting them. Both hosts have valid static addresses, but neither has a route to the other's subnet. That is the gap a router or gateway solves.

ARP observation

Temporarily moved VM2 Adapter 3 to labnet-a with a secondary address 10.10.10.30/24, then captured ARP traffic on VM1 while pinging it. Used:
sudo ip -s neigh flush all
sudo tcpdump -i enp0s9 -n arp
then from a second session: ping -c2 10.10.10.30

Lab 5.A, router

Topology

Windows host, bridged for SSH to all VMs. Host-only was unavailable due to a VirtualBox driver issue, so bridged was substituted for management access.

router ubuntu8: 10.10.10.1 on labnet-a, 10.10.20.1 on labnet-b
ubuntu4: 10.10.10.20 on labnet-a
ubuntu6: 10.10.20.20 on labnet-b

Router adapters

Adapter 1 bridged, for SSH
Adapter 2 disabled, host-only not working on this host
Adapter 3 internal network labnet-a
Adapter 4 internal network labnet-b

IP forwarding, persistent, survives reboot

/etc/sysctl.d/99-router.conf:
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1

After a full reboot:
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1

Checkpoint, bidirectional ping, IPv4 and IPv6

VM1 to VM2:
PING 10.10.20.20 (10.10.20.20) 56(84) bytes of data.
64 bytes from 10.10.20.20: icmp_seq=1 ttl=63 time=3.51 ms
64 bytes from 10.10.20.20: icmp_seq=2 ttl=63 time=1.70 ms
64 bytes from 10.10.20.20: icmp_seq=3 ttl=63 time=2.10 ms
3 packets transmitted, 3 received, 0% packet loss, time 2029ms

PING fd00:20::20 (fd00:20::20) 56 data bytes
64 bytes from fd00:20::20: icmp_seq=1 ttl=63 time=4.51 ms
64 bytes from fd00:20::20: icmp_seq=2 ttl=63 time=2.00 ms
64 bytes from fd00:20::20: icmp_seq=3 ttl=63 time=1.94 ms
3 packets transmitted, 3 received, 0% packet loss, time 2033ms

VM2 to VM1:
PING 10.10.10.20 (10.10.10.20) 56(84) bytes of data.
64 bytes from 10.10.10.20: icmp_seq=1 ttl=63 time=2.09 ms
64 bytes from 10.10.10.20: icmp_seq=2 ttl=63 time=1.69 ms
64 bytes from 10.10.10.20: icmp_seq=3 ttl=63 time=1.64 ms
3 packets transmitted, 3 received, 0% packet loss, time 5450ms

PING fd00:10::20 (fd00:10::20) 56 data bytes
64 bytes from fd00:10::20: icmp_seq=1 ttl=63 time=1.60 ms
64 bytes from fd00:10::20: icmp_seq=2 ttl=63 time=1.87 ms
64 bytes from fd00:10::20: icmp_seq=3 ttl=63 time=1.91 ms
3 packets transmitted, 3 received, 0% packet loss, time 3355ms

mtr from VM1, two hops

Start: 2026-10-05T06:03:09+0000
HOST: ubuntu4
  1. 10.10.10.1       0.0%    4    1.1   0.9   1.4   0.2
  2. 10.10.20.20       0.0%    4    1.9   2.0   2.2   0.1

Troubleshooting note: during setup a stray netplan file was briefly applied to the router that duplicated VM1's address onto the router's own labnet-a interface, causing an address conflict and loss of connectivity between VM1 and the router. Diagnosed bottom-up per the lab's method: ip link confirmed the interface was up, ip addr revealed the duplicate/secondary address and a failed IPv6 duplicate address detection, which identified the cause. Fixed by flushing the interface and reapplying netplan, confirmed persistent across a full reboot.

Lab 5.B, DNS, unbound

Resolver installed on the router. Config at /etc/unbound/unbound.conf.d/lab.conf:

server:
    interface: 10.10.10.1
    interface: 10.10.20.1
    access-control: 10.10.10.0/24 allow
    access-control: 10.10.20.0/24 allow
    access-control: 0.0.0.0/0 refuse
    local-zone: "lab.local." static
    local-data: "router.lab.local. IN A 10.10.10.1"
    local-data: "ubuntu.lab.local. IN A 10.10.10.20"
    local-data: "rocky.lab.local. IN A 10.10.20.20"

VM1 points at the resolver via Netplan nameservers: addresses: [10.10.10.1]
VM2 points at the resolver via nmcli ipv4.dns "10.10.20.1"

Caching evidence, two dig queries direct to the resolver

First query:
; <<>> DiG 9.20.24-1ubuntu0.3-Ubuntu <<>> example.com @10.10.10.1
;; status: NOERROR, id: 39074
;; ANSWER SECTION:
example.com.            300     IN      A       104.20.23.154
example.com.            300     IN      A       172.66.147.243
;; Query time: 847 msec
;; SERVER: 10.10.10.1#53(10.10.10.1) (UDP)

Second query:
; <<>> DiG 9.20.24-1ubuntu0.3-Ubuntu <<>> example.com @10.10.10.1
;; status: NOERROR, id: 40920
;; ANSWER SECTION:
example.com.            300     IN      A       104.20.23.154
example.com.            300     IN      A       172.66.147.243
;; Query time: 1 msec
;; SERVER: 10.10.10.1#53(10.10.10.1) (UDP)

Query time dropped from 847ms to 1ms on the second lookup, since the first query triggered a full recursive resolution out to the internet, and the second was served directly from unbound's cache.

Local names

Chose the unbound local-data zone approach over per-host /etc/hosts entries. A single entry on the resolver is visible to every host on both LANs automatically. /etc/hosts requires editing every machine individually and keeping them all in sync, which does not scale as the number of hosts grows and is prone to drift when one machine is updated and another is forgotten.

Test:
; <<>> DiG 9.20.24-1ubuntu0.3-Ubuntu <<>> ubuntu.lab.local @10.10.10.1
;; status: NOERROR, id: 4724
;; ANSWER SECTION:
ubuntu.lab.local.       3600    IN      A       10.10.10.20
;; SERVER: 10.10.10.1#53(10.10.10.1) (UDP)

Three failures table

Symptom: query returns REFUSED, SERVER shows 127.0.0.53, warning says .local is reserved for mDNS
Distinguishing command: resolvectl status, shows wrong DNS server 1.1.1.1 instead of 10.10.10.1
Cause: wrong resolver configured on the client
Fix: resolvectl dns enp0s9 10.10.10.1, or correct the Netplan nameservers block

Symptom: query returns REFUSED, minimal response with no question section, flags qr rd ad
Distinguishing command: dig ubuntu.lab.local @10.10.10.1 run directly against the resolver, status REFUSED despite the server being reachable
Cause: unbound access-control blocking the client's subnet
Fix: correct the access-control line in unbound.conf.d/lab.conf back to allow, restart unbound

Symptom: name resolves successfully but to the wrong IP, 10.10.20.99 instead of 10.10.10.20
Distinguishing command: cat /etc/hosts or getent hosts ubuntu.lab.local, shows the stale entry
Cause: /etc/hosts takes priority over DNS and had a stale manual entry
Fix: remove the stale line from /etc/hosts

Evidence, access-control refuse test

; <<>> DiG 9.20.24-1ubuntu0.3-Ubuntu <<>> ubuntu.lab.local @10.10.10.1
;; status: REFUSED, id: 8578
;; flags: qr rd ad; QUERY: 0, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 0
;; WARNING: recursion requested but not available
;; SERVER: 10.10.10.1#53(10.10.10.1) (UDP)

Evidence, stale /etc/hosts test

PING ubuntu.lab.local (10.10.20.99) 56(84) bytes of data.
From 10.10.10.1 icmp_seq=1 Destination Host Unreachable
From 10.10.10.1 icmp_seq=2 Destination Host Unreachable
2 packets transmitted, 0 received, +2 errors, 100% packet loss, time 1052ms

Evidence, wrong resolver test

Current DNS Server: 1.1.1.1
DNS Servers: 1.1.1.1

; <<>> DiG 9.20.24-1ubuntu0.3-Ubuntu <<>> ubuntu.lab.local
;; WARNING: .local is reserved for Multicast DNS
;; status: REFUSED, id: 15240
;; SERVER: 127.0.0.53#53(127.0.0.53) (UDP)

tcpdump capture of a DNS lookup, on VM1 enp0s9 port 53

06:24:36.293264 IP 10.10.10.20.46548 > 10.10.10.1.53: 4808+ [1au] A? ubuntu.lab.local. (57)
06:24:36.294566 IP 10.10.10.1.53 > 10.10.10.20.46548: 4808* 1/0/1 A 10.10.10.20 (61)
