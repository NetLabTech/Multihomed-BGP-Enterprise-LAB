# Multi-Vendor Network Lab (EVE-NG)

I wanted to build a network the way a real company would actually run one, instead of just doing little two-router labs. So this is my home lab in EVE-NG: two ISPs, a company edge, firewalls and a switched LAN behind them, all on different vendors (Juniper, Cisco and Fortinet). I also wanted to get properly hands-on with firewalls, so building them into the design was part of the point.

It's a work in progress and I'll keep adding to it. The roadmap at the bottom is what I'm doing next.

## The big picture

![Topology Diagram](https://github.com/NetLabTech/Multihomed-BGP-Enterprise-LAB/blob/41905f4efb90accf635ee4866b53716be25b700f/Screenshot%202026-10-11%20at%2014.25.25.png)

From the top down:

- **Two ISPs (Juniper vMX).** One is ISP A (AS 65100) and the other is ISP B (AS 65200). In real life these would sit in the providers' data centres, not in my building. Each one is a separate company with its own AS number, so my network is *multihomed* (connected to two providers).
- **The company edge (two Cisco CSR1000v routers).** This is AS 65001, my "company". Each CSR has a link to both ISPs, so any single cable, router or even a whole ISP can die and I still have a way out. In real life those four links would come into the comms room from the street through the ISP's demarc, ideally on different routes.
- **Two FortiGate firewalls.** One behind each CSR. Everything going between the inside and the outside has to pass through these. They're connected to each other by a dedicated heartbeat link for HA later on.
- **Two Cisco L3 switches (IOL).** One behind each firewall. This is the inside LAN where the desks and devices would plug in.

### Why the firewall sits in the middle

Routers decide *where* a packet goes. A firewall decides *whether it's allowed to go there at all*. If the switches plugged straight into the CSRs, everything inside would be sitting directly on the internet. Putting the firewall in the path means there's one checkpoint everything has to cross. The CSR/ISP side is "outside" (untrusted) and the switches/devices are "inside" (trusted). Later I'll make it stateful so the inside can start connections out and replies come back in, but anything that starts from outside gets dropped.

## Devices and images

| Role | Device | Image | Notes |
|---|---|---|---|
| ISP routers (brain) | vMX-VCP / vMX-VCP2 | vmxvcp-24.4R1.9 | The VCP is the control plane, all the config goes here |
| ISP routers (muscle) | vMX-VFP / vMX-VFP2 | vmxvfp-24.4R1.9 | The VFP just forwards packets, no CLI config on it |
| Company edge routers | CSR1 / CSR2 | Cisco CSR1000v (IOS-XE 17.3) | |
| Firewalls | Fortinet / Fortinet2 | FortiGate VM 7.4.12 | HA pair, not set up yet |
| L3 switch | CiscoSwitch1L3 | Cisco IOL `i86bi_LinuxL3-AdvEnterpriseK9-M2_157_3_May_2018` | |
| L3 switch | CiscoSwitch2L3 | Cisco IOL `i86bi_linux_l2-adventerprisek9-ms.SSA.high_iron_20180510` | IOS 15.2 build, name says L2 but it does L3 switching |

## IP addressing

There are three schemes and each has a reason.

**1. ISP links (CSR to vMX).** `/30`s from the documentation ranges `198.51.100.0/24` (ISP A) and `203.0.113.0/24` (ISP B). A `/30` has exactly two usable addresses, which is perfect for a cable with one device on each end. In real life an ISP hands you a small *public* block for the link. These ranges are reserved for docs and labs, so they act like public addresses without ever clashing with the real internet. It also keeps the outside looking different to the private `10.x` inside, which matters once I get to NAT.

| Link | vMX side | CSR side |
|---|---|---|
| ISP A: vMX ge-0/0/0 to CSR1 Gi1 | 198.51.100.1/30 | 198.51.100.2/30 |
| ISP A: vMX ge-0/0/1 to CSR2 Gi2 | 198.51.100.5/30 | 198.51.100.6/30 |
| ISP B: vMX2 ge-0/0/0 to CSR2 Gi1 | 203.0.113.1/30 | 203.0.113.2/30 |
| ISP B: vMX2 ge-0/0/1 to CSR1 Gi2 | 203.0.113.5/30 | 203.0.113.6/30 |

Other outside addresses:

- **My company's public block:** `192.0.2.0/24`. CSR1 Loopback100 is `192.0.2.1/32` and CSR2 Loopback100 is `192.0.2.2/32` (test hosts).
- **A pretend internet server:** `198.18.0.1/32` on lo0 of both ISPs, so I have something to ping.
- **ISP-to-ISP link** (vMX ge-0/0/9 to vMX2 ge-0/0/9): `10.1.1.17/30` and `10.1.1.18/30`. It stands in for the two providers connecting to each other.

**2. The firewall side** uses `/24`s from `10.10.0.0`. Bigger than a point-to-point link needs, but it leaves room to grow and keeps the security side visibly separate from the ISP side.

| Link | Side A | Side B |
|---|---|---|
| CSR1 Gi3 to FortiGate1 port1 | 10.10.1.1/24 | 10.10.1.2/24 |
| CSR2 Gi3 to FortiGate2 port1 | 10.10.2.1/24 | 10.10.2.2/24 |
| FortiGate1 port2 to FortiGate2 port2 (heartbeat) | 10.10.12.1/24 | 10.10.12.2/24 |
| FortiGate1 port3 to Switch1 e0/0 | 10.10.3.1/24 | 10.10.3.2/24 |
| FortiGate2 port3 to Switch2 e0/0 | 10.10.4.1/24 | 10.10.4.2/24 |

**3. Router IDs and loopbacks.** CSR1 is `1.1.1.1` and CSR2 is `2.2.2.2`. The ISPs use `198.18.1.1` (ISP A) and `198.18.2.1` (ISP B) as their router IDs, each on its own lo0 next to the shared `198.18.0.1`.

Every IP also has a text label on the EVE-NG diagram, and the `.unl` in this repo includes them.

## What I've configured so far

Config blocks are collapsed so the page doesn't get huge. Click to open them.

### BGP (the main thing so far)

My company (AS 65001) peers with both ISPs over eBGP. The rules I set up:

- **The ISPs only send me a default route** (`0.0.0.0/0`). That's how a normal customer connection works.
- **I only advertise my own block** (`192.0.2.0/24`). A prefix-list and route-map on the CSRs make sure I never accidentally pass one ISP's routes to the other, which would turn me into a transit network.
- **The ISPs only accept `192.0.2.0/24` from me**, the way a real provider filters its customers.
- **ISP A is my preferred way out.** The CSRs set local-preference 200 on routes from ISP A, and ISP B is the backup.
- **CSR1 is the preferred way in.** CSR2 advertises the block with a worse MED (100), so both ISPs send return traffic to CSR1 while it's up.
- My inside `10.10.x.x` networks are never advertised.

<details>
<summary>vMX-VCP (ISP A, AS 65100), Junos</summary>

```
set interfaces ge-0/0/0 unit 0 family inet address 198.51.100.1/30
set interfaces ge-0/0/1 unit 0 family inet address 198.51.100.5/30
set interfaces ge-0/0/9 unit 0 family inet address 10.1.1.17/30
set interfaces lo0 unit 0 family inet address 198.18.0.1/32
set interfaces lo0 unit 0 family inet address 198.18.1.1/32
set routing-options router-id 198.18.1.1
set routing-options autonomous-system 65100
set routing-options static route 0.0.0.0/0 discard

set policy-options prefix-list CUSTOMER-NETS 192.0.2.0/24
set policy-options policy-statement FROM-CUSTOMER term OK from prefix-list-filter CUSTOMER-NETS exact
set policy-options policy-statement FROM-CUSTOMER term OK then accept
set policy-options policy-statement FROM-CUSTOMER term REST then reject
set policy-options policy-statement TO-CUSTOMER term DEFAULT from protocol static
set policy-options policy-statement TO-CUSTOMER term DEFAULT from route-filter 0.0.0.0/0 exact
set policy-options policy-statement TO-CUSTOMER term DEFAULT then accept
set policy-options policy-statement TO-CUSTOMER term REST then reject

set protocols bgp group CUSTOMER type external
set protocols bgp group CUSTOMER peer-as 65001
set protocols bgp group CUSTOMER import FROM-CUSTOMER
set protocols bgp group CUSTOMER export TO-CUSTOMER
set protocols bgp group CUSTOMER neighbor 198.51.100.2
set protocols bgp group CUSTOMER neighbor 198.51.100.6
```
</details>

<details>
<summary>vMX-VCP2 (ISP B, AS 65200), Junos</summary>

Same as ISP A with different numbers:

```
set interfaces ge-0/0/0 unit 0 family inet address 203.0.113.1/30
set interfaces ge-0/0/1 unit 0 family inet address 203.0.113.5/30
set interfaces ge-0/0/9 unit 0 family inet address 10.1.1.18/30
set interfaces lo0 unit 0 family inet address 198.18.0.1/32
set interfaces lo0 unit 0 family inet address 198.18.2.1/32
set routing-options router-id 198.18.2.1
set routing-options autonomous-system 65200
set routing-options static route 0.0.0.0/0 discard
# same FROM-CUSTOMER / TO-CUSTOMER policies as ISP A
set protocols bgp group CUSTOMER type external
set protocols bgp group CUSTOMER peer-as 65001
set protocols bgp group CUSTOMER import FROM-CUSTOMER
set protocols bgp group CUSTOMER export TO-CUSTOMER
set protocols bgp group CUSTOMER neighbor 203.0.113.2
set protocols bgp group CUSTOMER neighbor 203.0.113.6
```
</details>

<details>
<summary>CSR1 (company edge, AS 65001), IOS-XE</summary>

```
interface Loopback100
 ip address 192.0.2.1 255.255.255.255
interface GigabitEthernet1
 ip address 198.51.100.2 255.255.255.252
 no shutdown
interface GigabitEthernet2
 ip address 203.0.113.6 255.255.255.252
 no shutdown
interface GigabitEthernet3
 ip address 10.10.1.1 255.255.255.0

ip route 192.0.2.0 255.255.255.0 Null0
ip prefix-list MY-NETS seq 5 permit 192.0.2.0/24
ip prefix-list DEFAULT-ONLY seq 5 permit 0.0.0.0/0

route-map TO-ISP permit 10
 match ip address prefix-list MY-NETS
route-map FROM-ISP-A permit 10
 match ip address prefix-list DEFAULT-ONLY
 set local-preference 200
route-map FROM-ISP-B permit 10
 match ip address prefix-list DEFAULT-ONLY

router bgp 65001
 bgp router-id 1.1.1.1
 bgp log-neighbor-changes
 neighbor 198.51.100.1 remote-as 65100
 neighbor 203.0.113.5 remote-as 65200
 address-family ipv4
  network 192.0.2.0 mask 255.255.255.0
  neighbor 198.51.100.1 activate
  neighbor 198.51.100.1 route-map TO-ISP out
  neighbor 198.51.100.1 route-map FROM-ISP-A in
  neighbor 203.0.113.5 activate
  neighbor 203.0.113.5 route-map TO-ISP out
  neighbor 203.0.113.5 route-map FROM-ISP-B in
```
</details>

<details>
<summary>CSR2 (company edge, AS 65001), IOS-XE</summary>

Same as CSR1, except:

```
interface Loopback100
 ip address 192.0.2.2 255.255.255.255
interface GigabitEthernet1
 ip address 203.0.113.2 255.255.255.252
interface GigabitEthernet2
 ip address 198.51.100.6 255.255.255.252
interface GigabitEthernet3
 ip address 10.10.2.1 255.255.255.0

route-map TO-ISP permit 10
 match ip address prefix-list MY-NETS
 set metric 100

router bgp 65001
 bgp router-id 2.2.2.2
 neighbor 198.51.100.5 remote-as 65100
 neighbor 203.0.113.1 remote-as 65200
 ! same address-family policies, applied to the two neighbours above
```
</details>

### Firewalls and switches

So far these just have their addressing (tables above) and I've pinged every link end to end. No firewall policy, NAT or HA yet, that's next on the roadmap. At the moment the FortiGates are really just passing traffic along.

## What I tested

- All four eBGP sessions come up, with each CSR peering with both ISPs.
- Both CSRs get a default route from both ISPs and pick ISP A as the best path.
- The CSRs only advertise `192.0.2.0/24`. The ISPs only advertise `0.0.0.0/0` and only accept `192.0.2.0/24` back.
- Both CSRs can ping the pretend internet server (`198.18.0.1`). From CSR1 I pinged it sourced from its `192.0.2.1` loopback three times, 5/5 every time.
- **Failover:** I shut the ISP A link on CSR1. The default route moved to ISP B and the ping carried on working. When I brought the link back, ISP A took over again.
- I shut the whole lab down and booted it again, and all the sessions came back up on their own with the config intact.

### Things I know are not finished

- `192.0.2.0/24` currently points at `Null0` on the CSRs. BGP advertises it fine, but traffic coming *in* to that block isn't forwarded to the inside yet. It needs the firewall policy and NAT.
- CSR1 and CSR2 don't have iBGP between them yet, so inbound is "CSR1 primary, CSR2 backup" and not both at once.

## vMX wiring (the bit that trips everyone up)

A vMX is two VMs holding hands: the VCP (brain) and the VFP (muscle). Before any `ge-` interfaces exist, you cable the internal links:

- VCP `em0` (fxp0) to VFP `em0` (fxp0): management and control
- VCP `em1` (internal) to VFP `em1` (internal): brain-to-muscle link
- VCP `em2` and up become `ge-0/0/0`, `ge-0/0/1`, and so on

```
root> show chassis fpc
```

Slot 0 must say `State: Online` before you configure anything. All config lives on the VCP, and the VFP console is a bare Linux login that you leave alone.

## Troubleshooting log (aka the graveyard)

Things that broke and what I learned:

- **vJunosEvo and Cisco vIOS: 0% ping both ways.** Addressing was right, interfaces were up/up, but ARP never completed. Swapping NIC types (virtio vs e1000) made no difference. I narrowed it down to vJunosEvo's dataplane, not the host, because vMX to CSR1000v on the same host works fine.
- **vMX and CSR1000v: clean.** Direct Junos-to-Junos pings at 100%, which proved the host bridging was fine.
- **Changing a NIC type needs a full stop and start of both ends of the link.** It's a qemu setting, not something the guest OS picks up live. IOSv breaks hard if you change it while running (empty `show ip int br`, unrecoverable), so recovery meant putting the NIC type back and/or an EVE-NG "Wipe".
- **FortiGate console from automation:** netmiko's `fortinet` driver is SSH only these days. Pointed at a telnet console it just garbles the login, so I use a raw telnet script.
- **The FortiGate console logs you out after 5 minutes by default.** `config system global` then `set admintimeout 480` stretches it.
- **Both CSRs advertising the same block to `Null0` made pings flaky.** Each ISP picks one CSR and sends all the return traffic there. If that wasn't the CSR that sent the ping, the packet hit `Null0` and died. Setting a worse MED on CSR2 makes it predictable. The real fix is iBGP plus a real route to the inside.
- **Two routers with the same BGP router-ID.** Both ISPs picked up `198.18.0.1` from the shared loopback. I gave each its own loopback and set the router-ID explicitly. A BGP session only sends the new ID after it resets.
- **A restored link didn't bring BGP back by itself** during the failover test. The first reconnect got rejected and I had to `clear bgp neighbor` on the vMX.
- **Junos keeps the oldest route when two are equal.** After a failover the return path can stay pinned to the "wrong" CSR until the session is cleared.

## Roadmap

Where I'm taking this next, roughly in order:

**Firewall and security**
- [ ] Firewall policies on the FortiGates: inside to outside allowed, outside to inside blocked
- [ ] Source NAT on the FortiGates so the `10.10.x.x` inside gets out on the public block `192.0.2.0/24`
- [ ] Change the `Null0` route on the CSRs to point at the firewall so inbound traffic is actually delivered
- [ ] Bring up FortiGate HA properly over the heartbeat link, then force a failover under load
- [ ] ACLs on the CSRs and the switches (keep the management plane locked down)
- [ ] Site-to-site IPsec VPN between the two sides of the network

**Routing**
- [ ] OSPF inside the company (CSRs, FortiGates and the L3 switches), keeping the ISP edge as BGP only
- [ ] iBGP between CSR1 and CSR2 so both can carry inbound traffic
- [ ] More BGP path control: AS-path prepending, communities, and filtering what the CSRs accept
- [ ] Redistribute between OSPF and BGP at the edge and watch for routing loops
- [ ] Route-maps and prefix-lists for policy-based routing between security zones

**The LAN**
- [ ] VLANs and SVIs on the L3 switches, with inter-VLAN routing
- [ ] DHCP and some test hosts on the VLANs
- [ ] EtherChannel between the switches and redundant links to the firewalls

**Operations**
- [ ] Basic QoS (classification and marking) on the CSRs
- [ ] NTP, syslog and SNMP so the lab behaves like a monitored network
- [ ] SSH only, no telnet, and proper login banners and local accounts
- [ ] Add a cloud/NAT object above the ISPs for real external NTP/DNS
- [ ] Export baseline configs for every device so I can roll back

## Using the lab file

`MESH.unl` is the actual EVE-NG lab. To import it:

1. Easiest: in the EVE-NG web UI, **Labs → Upload** (or **Import**) and pick `MESH.unl`. Permissions get handled for you.
2. Or copy it into `/opt/unetlab/labs/` and make sure the web server can read it: `chown www-data:www-data` and `chmod 644`.
3. You'll still need the images yourself (vMX 24.4R1.9, CSR1000v 17.3, FortiGate 7.4.12, IOL). They can't ship with the repo for licensing reasons.

The lab boots all 10 nodes at once, so give the host a decent amount of RAM. The device configs aren't stored in the `.unl`, so an imported lab starts blank. The config blocks above are what I applied.blocks above are what I applied.
