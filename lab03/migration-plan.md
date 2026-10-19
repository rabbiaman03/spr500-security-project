# SPR500 Lab 03 — Configure Network Segregation Policies: Migration Plan

**Group 6** · `group6.local` · Course: SPR500 Network Security

# 1. Purpose and scope

This plan migrates the Group 6 network from the Lab 2 design (Figure 1: VLAN segmentation, OSPF, stateless zone firewalls) to the Lab 3 design (Figure 2: an internal stateful L3 firewall at the Core). Traffic between security zones is segregated with VLANs, routing policies (OSPF area 0) and a stateful L3 firewall; the split-DNS pattern is completed; and every host firewall is configured from code with Ansible.

## What changes from Lab 2 to Lab 3

| **Area** | **Lab 2 (current)** | **Lab 3 (target)** |
|----|----|----|
| Internal (Core) firewall | Stateless nftables: every flow needs a separate reply rule (tcp flags != syn) | STATEFUL nftables: connection tracking returns replies automatically; invalid packets dropped; a NEW TCP flow must start with SYN |
| Rule structure | One long forward chain | Verdict map (in-interface . out-interface) jumps to one policy chain per zone pair; addresses and ports in named sets |
| Advanced capabilities | – | Layer-7 string matching (blocks “GET /admin”, “POST /admin”, “TRACE” from the Outside); per-source SYN-flood meter |
| Host firewalls | Hand-written on some Linux servers; Windows defaults | Default-deny host firewall on every server and PC, generated and deployed by Ansible |
| Firewall automation | None (configs copied by hand) | Ansible control node “mgmt” 192.168.6.131 in the Admin VLAN; one playbook deploys router and host firewalls, syntax-checked (nft -c) before install |
| Management access | Admin PC 192.168.6.130 | Admin PC 192.168.6.130 + Ansible node 192.168.6.131 (SSH 22 to Linux, WinRM 5985 to Windows) |
| Split DNS | Private and public DNS servers | Unchanged design, now also enforced by each DNS server's own host firewall (private: internal subnets only; public: any) |
| User PC address | DHCP pool .50 – .120 | DHCP reservation 192.168.6.40 (stable target for Ansible) |

# 2. Network design diagram

![](images/media/image1.png)

*Figure 1 – Lab 3 target network: Edge → stateful Core → Open vSwitch VMs, host firewalls on every server/PC (red badge), Ansible node in the Admin VLAN*

# 3. Business requirements

**Table 1 – List of Business requirements**

| **Req_ID** | **Requirement description** | **Test case description** | **Acceptance criteria** |
|----|----|----|----|
| BR-01 | Employees use the internal web application (Odoo) and the file share | From the User PC open http://intweb.group6.local and \\\172.16.6.133\share | Odoo login page loads; share lists readme.txt |
| BR-02 | Customers on the Internet use the public website and the online store | From OutsideUser open http://corpweb.group6.local and http://esite.group6.local | Joomla and PrestaShop pages load |
| BR-03 | Internal systems are not reachable from the Internet | From Kali: nmap -Pn 172.16.6.132 172.16.6.133 10.6.6.10 192.168.6.130 | All ports filtered |
| BR-04 | Split DNS: private DNS answers only authorised internal endpoints; public DNS answers any endpoint about public assets only | nslookup intweb from User (10.6.6.12) and from OutsideUser (172.16.6.53); query 10.6.6.12 from Outside | Internal: answer; Outside: NXDOMAIN for intweb, timeout on 10.6.6.12 |
| BR-05 | Central authentication with Active Directory | Log on to the User PC as GROUP6\employee1; nltest /dsgetdc:group6.local | Logon succeeds; DC = dc.group6.local |
| BR-06 | Clients receive addresses automatically across VLANs | ipconfig /renew on the User PC | 192.168.6.40 from DHCP server 10.6.6.11 via relay 192.168.6.1 |
| BR-07 | Only administrators can manage network devices and servers | SSH to the Core from the User PC and from the Admin PC / Ansible node | User: blocked; Admin/Ansible: login prompt |
| BR-08 | Traffic between security zones passes a stateful L3 firewall | nft list chain inet filter forward on the Core; nmap -sA (ACK scan) from Kali | ct state rules present; ACK scan shows filtered; CORE-NEW-NOT-SYN logged |
| BR-09 | Every host firewall is configured with Ansible | ansible-playbook site.yml --check on mgmt; Get-NetFirewallProfile on Windows | 0 pending changes; default inbound = Block |
| BR-10 | Dynamic, authenticated routing (no static routes) | show ip ospf neighbor / show ip route on the Core; stop FRR on Edge | Neighbour Full, MD5; default route removed in ≤ 20 s and relearned |
| BR-11 | Web administration portals are not exposed to the Internet | From Outside request /administrator and /admin-g6; from Admin VLAN request /administrator | Outside: dropped (CORE-L7-ADMIN-BLOCK); inside: HTTP 200 |
| BR-12 | Denied traffic is logged for investigation | journalctl -k on Core / Edge / hosts during blocked tests | CORE-\*, EDGE-\*, HOSTFW-DROP log lines with source/destination |
| BR-13 | Staff have Internet access | From the User PC open https://www.seneca.ca | Page loads through Edge NAT |

# 4. Network devices

**Table 2 – List of Network Devices**

| **FQDN** | **Role** | **Interface** | **Old IP (CIDR)** | **New IP (CIDR)** | **Old gateway** | **New gateway** | **VLAN** |
|----|----|----|----|----|----|----|----|
| edge.group6.local | Edge router, perimeter FW, NAT | wan0 / out0 / tr0 | DHCP / 10.10.10.1/24 / 192.168.6.145/30 | unchanged | VMware NAT | VMware NAT | – / Outside / Transit |
| core.group6.local | Core router, STATEFUL zone FW, DHCP relay | tr0 | 192.168.6.146/30 | unchanged | OSPF default via .145 | OSPF default via .145 | Transit |
|  |  | vl10 / vl20 / vl30 / vl40 / vl50 (on trunk0) | 192.168.6.1/25 · 192.168.6.129/28 · 172.16.6.1/25 · 172.16.6.129/25 · 10.6.6.1/24 | unchanged | – | – | 10 / 20 / 30 / 40 / 50 |
| sw1 / sw2 / sw3 | Open vSwitch access/distribution switches | uplink, to-sw1, to-sw3, p-\* ports | no IP (L2) | no IP; SW1 adds port p-mgmt (VLAN 20) | – | – | trunks 10-50 |
| user.group6.local | Employee PC (Windows 10) | Ethernet0 | DHCP 192.168.6.50-120/25 | 192.168.6.40/25 (DHCP reservation) | 192.168.6.1 | 192.168.6.1 | 10 |
| admin.group6.local | Administrator PC (Windows 10) | Ethernet0 | 192.168.6.130/28 | unchanged | 192.168.6.129 | 192.168.6.129 | 20 |
| mgmt.group6.local | NEW – Ansible control node (Ubuntu) | adm0 | – (new) | 192.168.6.131/28 | – | 192.168.6.129 | 20 |
| corpweb / esite.group6.local | Docker host + Joomla, PrestaShop containers | dmz0 (macvlan) | 172.16.6.10, .11, .14 /25 | unchanged | 172.16.6.1 | 172.16.6.1 | 30 |
| pubdns.group6.local | Public DNS (BIND) | dmz0 | 172.16.6.53/25 | unchanged | 172.16.6.1 | 172.16.6.1 | 30 |
| intweb / files.group6.local | Docker host + Odoo, Samba containers | app0 (macvlan) | 172.16.6.130, .132, .133 /25 | unchanged | 172.16.6.129 | 172.16.6.129 | 40 |
| dc.group6.local | Active Directory DS + AD DNS | Ethernet0 | 10.6.6.10/24 | unchanged | 10.6.6.1 | 10.6.6.1 | 50 |
| netsvc.group6.local | DHCP server | svc0 | 10.6.6.11/24 | unchanged | 10.6.6.1 | 10.6.6.1 | 50 |
| privdns.group6.local | Private DNS (BIND) | svc0 | 10.6.6.12/24 | unchanged | 10.6.6.1 | 10.6.6.1 | 50 |
| outsideuser | Internet customer (Windows 10) | Ethernet0 | 10.10.10.2/24 | unchanged | 10.10.10.1 | 10.10.10.1 | Outside |
| attacker | External attacker (Kali) | eth0 | 10.10.10.3/24 | unchanged | 10.10.10.1 | 10.10.10.1 | Outside |

# 5. Software

**Table 3 – List of Software**

| **FQDN** | **Role** | **OS** | **Software** | **Version** |
|----|----|----|----|----|
| edge, core | Routers / network firewalls | Ubuntu 24.04 LTS (kernel 6.8) | FRRouting (OSPF) · nftables · isc-dhcp-relay (core) | 8.4.4 · 1.0.9 · 4.4.3 |
| sw1, sw2, sw3 | Virtual switches | Ubuntu 24.04 LTS | Open vSwitch | 3.3.9 |
| mgmt | Firewall automation | Ubuntu 24.04 LTS | Ansible core · ansible.windows · community.windows · pywinrm | 2.16.3 · 2.2.0 · 2.1.0 |
| corpweb / esite host | Public web applications | Ubuntu 24.04 LTS | Docker · Joomla (joomla:5-apache) · PrestaShop 8 · MariaDB 11 · nftables host FW | 29.1.3 · 5.4.9 · 8.2.8 · 11 · 1.0.9 |
| intweb / files host | Internal applications | Ubuntu 24.04 LTS | Docker · Odoo · PostgreSQL · Samba (dperson/samba) · nftables host FW | 29.1.3 · 17 · 16 · latest · 1.0.9 |
| pubdns, privdns | Split DNS | Ubuntu 24.04 LTS | BIND 9 · nftables host FW | 9.18.39 · 1.0.9 |
| netsvc | DHCP | Ubuntu 24.04 LTS | isc-dhcp-server · nftables host FW | 4.4.3-P1 · 1.0.9 |
| dc | Directory services | Windows Server 2019 Datacenter (17763) | AD DS · AD DNS · Windows Defender Firewall · WinRM | 2019 |
| user, admin, outsideuser | Desktops | Windows 10 Pro 22H2 (19045) | Windows Defender Firewall · WinRM (user, admin) | 22H2 |
| attacker | Security testing | Kali Linux | nmap · nikto · dig | rolling |
| host PC | Virtualisation | Windows 11 | VMware Workstation Pro | 17.6 |

# 6. Security zones

**Table 4 – List of Security Zones**

| **Security zone name** | **Description** | **Members (network devices)** |
|----|----|----|
| Outside (untrusted) | Internet customers and attackers; 10.10.10.0/24 | outsideuser, attacker, edge out0 |
| Transit | Routed Edge–Core link, OSPF area 0 with MD5; 192.168.6.144/30 | edge tr0, core tr0 |
| DMZ – VLAN 30 | Internet-facing services; 172.16.6.0/25 | corpweb (Joomla), esite (PrestaShop), pubdns, Docker host .10 |
| Users – VLAN 10 | Employee workstations; 192.168.6.0/25 | user |
| Admin / Management – VLAN 20 | Administration and automation; 192.168.6.128/28 | admin, mgmt (Ansible) |
| Internal – VLAN 40 | Internal business applications; 172.16.6.128/25 | intweb (Odoo), files (Samba), Docker host .130 |
| Infrastructure – VLAN 50 | Core services; 10.6.6.0/24 | dc (AD), netsvc (DHCP), privdns |
| Network core | Routing and the stateful zone firewall between all zones | core, sw1, sw2, sw3 |

# 7. Network access policies

**Table 5 – Network Access Polices**

| **From device** | **To device** | **Protocol** | **Port** | **Reason** |
|----|----|----|----|----|
| user, admin | corpweb .11, esite .14 | TCP | 80, 443 | Use the public website and store |
| user, admin | pubdns .53 | TCP/UDP | 53 | Resolve public names |
| user, admin | intweb .132 | TCP | 80, 443 | Internal Odoo application |
| user, admin | files .133 | TCP | 445 | File sharing (SMB) |
| user, admin | privdns .12 | TCP/UDP | 53 | Private DNS (internal names, Internet recursion) |
| user, admin | dc .10 | TCP 88,135,389,445,464,636,3268-3269,49152-65535 / UDP 88,123,389,464 | as listed | Domain logon, Kerberos, LDAP, GPO, time |
| user | netsvc .11 (via core relay) | UDP | 67/68 | DHCP |
| admin .130 | dc .10 | TCP | 3389 | Administer the domain controller (RDP) |
| admin .130 | user | TCP | 3389 | Remote support |
| admin .130, mgmt .131 | core, edge, Linux servers | TCP | 22 | SSH management / Ansible |
| mgmt .131 | dc, admin, user | TCP | 5985 | WinRM – Ansible configures Windows Firewall |
| outsideuser, attacker | corpweb .11, esite .14 | TCP | 80, 443 | Public website and store (admin paths blocked by L7 rule) |
| outsideuser, attacker | pubdns .53 | TCP/UDP | 53 | Public DNS |
| intweb/files host | privdns .12 | TCP/UDP | 53 | Name resolution for internal apps |
| Docker hosts, DNS servers, dc, netsvc | Internet | TCP 53/80/443, UDP 53/123 | as listed | Updates, image pulls, DNS forwarders, NTP |
| user, admin | Internet / Outside | any | any | Staff Internet access (NAT on edge) |
| core | edge | OSPF (IP 89) | – | Dynamic routing |
| Any other combination | – | – | – | Denied and logged (default deny) |

# 8. Security zone boundaries

**Table 6 – Security Zone Boundaries**

| **From security zone** | **To security zone** | **Protocol** | **Reason** |
|----|----|----|----|
| Outside | DMZ | HTTP/HTTPS to web, DNS to pubdns | Publish only the public services; /admin and TRACE dropped by string matching; SYN-flood meter |
| Outside | Users / Admin / Internal / Infra | none (denied) | Internal zones are never reachable from the Internet |
| Users | DMZ | HTTP/HTTPS, DNS | Employees use the public apps |
| Users | Internal | HTTP/HTTPS, SMB | Odoo and file sharing |
| Users | Infra | DNS (privdns), AD protocols, DHCP via relay | Logon, name resolution, addressing; no direct DNS to the DC |
| Users | Admin | none (denied, logged CORE-USR-ADMIN-DROP) | A compromised user PC cannot reach the administrators |
| Admin | all internal zones | Staff services + SSH, RDP (Admin PC), WinRM (Ansible) | Administration and firewall automation |
| Users / Admin | Outside / Internet | any (stateful, replies only back) | Internet access through NAT |
| Internal | Infra | DNS to privdns | Name resolution only |
| DMZ / Internal / Infra | Internet | DNS, HTTP/HTTPS, NTP from listed servers | Updates and forwarders |
| DMZ | Users / Admin / Internal / Infra | none (denied) | A compromised DMZ server cannot pivot inwards |
| Inside one VLAN (host to host) | same VLAN | only the service ports each host publishes | Host firewalls stop lateral movement the Core never sees |

# 9. Firewall test cases

**Table 7 – List of Firewall Test Cases**

| **Sec_Req_ID** | **Network device** | **Firewall policies** | **Test case description** | **Acceptance criteria** |
|----|----|----|----|----|
| SR-01 | core | forward: ct state established,related accept; ct state invalid drop | nft list chain inet filter forward | Stateful rules present; policy drop |
| SR-02 | core | ct state new + TCP flags != SYN → drop | Kali: sudo nmap -sA -Pn 172.16.6.11 | Ports reported filtered; CORE-NEW-NOT-SYN in log |
| SR-03 | core | web_inspect: @ih string match “GET /admin”, “POST /admin”, “TRACE” | Outside: curl http://172.16.6.11/administrator/ and /admin-g6/; Admin VLAN: same URL | Outside: no response, CORE-L7-ADMIN-BLOCK; inside: HTTP 200 |
| SR-04 | core | outside_to_dmz: SYN-flood meter 25/s per source | nft list chain inet filter outside_to_dmz | syn_flood set and CORE-SYNFLOOD rule present |
| SR-05 | core | users_to_admin: drop + log | User PC: Test-NetConnection 192.168.6.130 -Port 3389 | False; CORE-USR-ADMIN-DROP |
| SR-06 | core | anti-spoofing per VLAN | User PC with static 192.168.6.130 → ssh 192.168.6.129 | Fails; CORE-SPOOF-DROP IN=vl10 |
| SR-07 | core + edge | Outside → DMZ web/DNS only | Kali: nmap -sV -Pn 172.16.6.11 172.16.6.14 172.16.6.53 | Only 80 and 53 open |
| SR-08 | core + edge | Outside → internal zones denied | Kali: nmap -Pn 172.16.6.132 172.16.6.133 10.6.6.10 192.168.6.130 192.168.6.131 | All filtered; EDGE-FWD-DROP |
| SR-09 | privdns host FW | 53 only from internal subnets | dig @10.6.6.12 from Outside / from User | Outside: timeout; User: answer |
| SR-10 | pubdns host FW | 53 from any; no recursion | dig @172.16.6.53 esite / ubuntu.com from Outside | Answer for esite; none for ubuntu.com |
| SR-11 | netsvc, dc host FW | SSH/RDP only from admin hosts (same VLAN) | From privdns: connect 10.6.6.11:22 and 10.6.6.10:3389 | Both closed (traffic never crosses the Core) |
| SR-12 | all Linux hosts | table inet hostfw, policy drop, SSH from admin_hosts | ansible linux_servers -b -m shell -a 'nft list chain inet hostfw input' | policy drop on every host; g6-hostfw active |
| SR-13 | dc, admin, user | Windows Firewall: all profiles on, inbound Block, G6-Lab3 rules | Get-NetFirewallProfile; Get-NetFirewallRule -Group G6-Lab3 | Enabled, DefaultInboundAction Block, scoped rules |
| SR-14 | mgmt | Ansible idempotency | ansible-playbook site.yml --check | changed=0 on every reachable host |
| SR-15 | core, edge | OSPF MD5 between routers | show ip ospf interface tr0; stop/start frr on edge | MD5 on; neighbour lost \< 20 s and restored |

# 10. Migration steps and rollback

| **Step** | **Action** | **Verification** | **Rollback** |
|----|----|----|----|
| 0 | Snapshot every VM as “Lab2-final” | 15 snapshots listed in VMware | Revert to Lab2-final |
| 1 | Create mgmt VM (Ubuntu), install Ansible + pywinrm; add SW1 port p-mgmt (VLAN 20) and LAN segment LAB02-P-SW1-Mgmt | mgmt pings 192.168.6.129 | Power off mgmt; remove SW1 adapter 4 |
| 2 | Apply the stateful Core firewall and the Edge rule allowing SSH from mgmt (once by hand, then Ansible-managed) | nft -c passes; OSPF Full; tests from all zones | cp /etc/nftables.conf.lab2 /etc/nftables.conf; nft -f |
| 3 | Install the Ansible project (inventory, roles router_fw, linux_hostfw, windows_hostfw; passwords in ansible-vault) | ansible linux -m ping; ansible windows -m win_ping | – |
| 4 | Bootstrap SSH (netsvc) and WinRM (dc, admin, user) limited to 192.168.6.131 | Ansible reaches all hosts | Disable WinRM / remove G6 rule |
| 5 | DHCP reservation 192.168.6.40 for the User PC | DHCPACK 192.168.6.40 | Remove host user-pc block |
| 6 | ansible-playbook site.yml (routers → Linux hosts → Windows hosts) | PLAY RECAP failed=0 | Old files kept: /etc/nftables.conf.lab2; Windows rules group G6-Lab3 |
| 7 | Run tests-lab3.ps1 and the User/Admin/Outside test cases | All PASS | Revert snapshot |

# 11. Logbook

**Table 8 – Logbook (maximum task duration 30 minutes)**

| **From (date time)** | **To (date time)** | **Task description and reason** | **Evaluation and justification** | **Req_ID** | **Task owner** |
|----|----|----|----|----|----|
| 2026-10-19 \_\_:\_\_ | 2026-10-19 \_\_:\_\_ | Read Lab 3 requirements, compare Figure 1 and Figure 2, list changes | Changes: stateful L3 FW, host FW automation, split DNS kept | all |  |
| 2026-10-19 \_\_:\_\_ | 2026-10-19 \_\_:\_\_ | Snapshot all 15 VMs as Lab2-final | Rollback point before any change | all |  |
| 2026-10-19 \_\_:\_\_ | 2026-10-19 \_\_:\_\_ | Clone mgmt VM, install Ansible 2.16 + Windows collections + pywinrm | Control node for firewall automation | BR-09 |  |
| 2026-10-19 \_\_:\_\_ | 2026-10-19 \_\_:\_\_ | Add SW1 port p-mgmt (VLAN 20), new LAN segment, static IP 192.168.6.131 | Ansible node sits in the Admin VLAN | BR-07 |  |
| 2026-10-19 \_\_:\_\_ | 2026-10-19 \_\_:\_\_ | Write the stateful Core firewall (zone-pair chains, sets, L7 string match, SYN meter); nft -c | Syntax check passed on the Core | BR-08, BR-11 |  |
| 2026-10-19 \_\_:\_\_ | 2026-10-19 \_\_:\_\_ | Apply Core and Edge firewalls; verify OSPF and default route | Neighbour Full, routes unchanged | BR-10 |  |
| 2026-10-19 \_\_:\_\_ | 2026-10-19 \_\_:\_\_ | Write Ansible inventory, group/host vars, roles router_fw and linux_hostfw | Templates validated with nft -c before install | BR-09 |  |
| 2026-10-19 \_\_:\_\_ | 2026-10-19 \_\_:\_\_ | Bootstrap SSH on netsvc; deploy Linux host firewalls; retire hand-written Lab 2 host rules | 6 hosts: g6-hostfw active, policy drop; Docker rules intact | BR-09 |  |
| 2026-10-19 \_\_:\_\_ | 2026-10-19 \_\_:\_\_ | Enable WinRM (Mgmt only) on dc, admin, user; write windows_hostfw role | win_ping OK on all three | BR-09 |  |
| 2026-10-19 \_\_:\_\_ | 2026-10-19 \_\_:\_\_ | Deploy Windows host firewalls; DHCP reservation 192.168.6.40 for the User PC | Inbound Block on all profiles; failed=0 | BR-06, BR-09 |  |
| 2026-10-19 \_\_:\_\_ | 2026-10-19 \_\_:\_\_ | Add NEW-must-be-SYN rule and push it through Ansible (--diff) | Diff shows one line; Core updated by automation | BR-08 |  |
| 2026-10-19 \_\_:\_\_ | 2026-10-19 \_\_:\_\_ | Run User PC tests and tests-lab3.ps1 | 20/20 user checks, 45/45 lab checks pass | all |  |
| 2026-10-19 \_\_:\_\_ | 2026-10-19 \_\_:\_\_ | Update the network diagram and write this migration plan | Diagram and tables match the deployed system | all |  |

# Appendix A – Ansible project (mgmt:~sysadmin/g6-firewall)

site.yml playbook: routers → linux_servers → windows\
inventory.ini core, edge \| webpublic, publicdns, appsinternal, dhcp, privatedns, mgmt \| ad, admin-pc, user-pc\
group_vars/all/main.yml admin_hosts, internal_nets, mgmt_node (one place to change addresses)\
group_vars/all/vault.yml passwords (ansible-vault AES256)\
roles/router_fw core/edge nftables.conf, validated with 'nft -c' before install, backup kept\
roles/linux_hostfw Jinja2 template -\> /etc/g6-hostfw.nft (table inet hostfw) + g6-hostfw.service\
roles/windows_hostfw scoped G6-Lab3 rules, firewall on with inbound Block, broad built-in rules retired\
\
Run: ansible-playbook site.yml (deploy everything)\
ansible-playbook site.yml --check --diff (show what would change)\
ansible-playbook site.yml --limit core (one device)

# Appendix B – Core stateful firewall (excerpt)

\#!/usr/sbin/nft -f\
\# CORE router - group6.local - Lab 3 STATEFUL L3 zone firewall (managed by Ansible - do not edit by hand)\
\# tr0 = Transit to Edge 192.168.6.144/30 (Outside + Internet behind it)\
\# vl10 = Users 192.168.6.0/25 vl20 = Admin 192.168.6.128/28\
\# vl30 = DMZ 172.16.6.0/25 vl40 = Internal 172.16.6.128/25 vl50 = Infra 10.6.6.0/24\
\# Design: conntrack lets replies back in automatically; every NEW flow is decided in one chain per zone pair,\
\# chosen by a verdict map (iif . oif). Addresses and ports live in named sets so a change is one line.\
flush ruleset\
define USERS = 192.168.6.0/25\
define ADMINNET = 192.168.6.128/28\
define DMZ = 172.16.6.0/25\
define INTAPPS = 172.16.6.128/25\
define INFRA = 10.6.6.0/24\
define OUTSIDE = 10.10.10.0/24\
define AD = 10.6.6.10\
define DHCPSRV = 10.6.6.11\
define PRIVDNS = 10.6.6.12\
define PUBDNS = 172.16.6.53\
define INTWEB = 172.16.6.132\
define FILESRV = 172.16.6.133\
define MGMT = 192.168.6.131\
table inet filter {\
\# ---------- named sets (maintainability: edit here, not in the rules) ----------\
set admins { type ipv4_addr; elements = { 192.168.6.130, 192.168.6.131 } } \# Admin PC + Ansible node\
set pubweb { type ipv4_addr; elements = { 172.16.6.11, 172.16.6.14 } } \# Joomla, PrestaShop\
set linux_srv { type ipv4_addr; elements = { 172.16.6.10, 172.16.6.53, 172.16.6.130, 10.6.6.11, 10.6.6.12 } }\
set inet_srv { type ipv4_addr; elements = { 172.16.6.10, 172.16.6.53, 172.16.6.130, 10.6.6.10, 10.6.6.11, 10.6.6.12 } }\
set lab_nets { type ipv4_addr; flags interval; elements = { 192.168.6.0/24, 172.16.6.0/24, 10.6.6.0/24, 10.10.10.0/24 } }\
set web_ports { type inet_service; elements = { 80, 443 } }\
set ad_tcp { type inet_service; flags interval; elements = { 88, 135, 389, 445, 464, 636, 3268, 3269, 49152-65535 } }\
set ad_udp { type inet_service; elements = { 88, 123, 389, 464 } }\
set syn_flood { type ipv4_addr; flags dynamic, timeout; timeout 1m; } \# per-source SYN meter\
chain input {\
type filter hook input priority filter; policy drop;\
iifname "lo" accept\
ct state invalid counter drop\
ct state established,related accept\
iifname "tr0" ip protocol ospf accept\
iifname "vl10" udp sport 68 udp dport 67 accept \# DHCP clients -\> relay\
iifname "vl50" ip saddr \$DHCPSRV udp dport 67 accept \# DHCP server -\> relay\
iifname "vl20" ip saddr @admins tcp dport 22 ct state new accept \# SSH: Admin PC + Ansible, only via VLAN 20\
icmp type echo-request limit rate 10/second accept\
log prefix "CORE-IN-DROP: " counter drop\
}\
chain forward {\
type filter hook forward priority filter; policy drop;\
\# 1) Layer-7 string matching on Outside -\> DMZ web traffic (must run before the established fast path)\
iifname "tr0" oifname "vl30" tcp dport 80 jump web_inspect\
\# 2) Stateful fast path: replies of flows that were allowed below\
ct state established,related counter accept\
ct state invalid log prefix "CORE-INVALID: " counter drop\
ct state new tcp flags & (fin \| syn \| rst \| ack) != syn log prefix "CORE-NEW-NOT-SYN: " counter drop \# stops ACK/FIN/Xmas scans\
\# 3) Anti-spoofing: every VLAN may only use its own subnet\
iifname "vl10" ip saddr != \$USERS log prefix "CORE-SPOOF-DROP: " counter drop\
iifname "vl20" ip saddr != \$ADMINNET log prefix "CORE-SPOOF-DROP: " counter drop\
iifname "vl30" ip saddr != \$DMZ log prefix "CORE-SPOOF-DROP: " counter drop\
iifname "vl40" ip saddr != \$INTAPPS log prefix "CORE-SPOOF-DROP: " counter drop\
iifname "vl50" ip saddr != \$INFRA log prefix "CORE-SPOOF-DROP: " counter drop\
iifname "tr0" ip saddr { \$USERS, \$ADMINNET, \$DMZ, \$INTAPPS, \$INFRA } log prefix "CORE-SPOOF-DROP: " counter drop\
\# 4) New connections: one policy chain per zone pair\
iifname . oifname vmap {\
"vl10" . "vl30" : jump staff_to_dmz, "vl20" . "vl30" : jump staff_to_dmz,\
"vl10" . "vl40" : jump staff_to_internal, "vl20" . "vl40" : jump staff_to_internal,\
"vl10" . "vl50" : jump staff_to_infra, "vl20" . "vl50" : jump staff_to_infra,\
"vl10" . "tr0" : jump staff_to_outside, "vl20" . "tr0" : jump staff_to_outside,\
"vl20" . "vl10" : jump admin_to_users,\
"vl10" . "vl20" : jump users_to_admin,\
"tr0" . "vl30" : jump outside_to_dmz,\
"vl40" . "vl50" : jump internal_to_infra,\
"vl30" . "tr0" : jump servers_to_internet,\
"vl40" . "tr0" : jump servers_to_internet,\
"vl50" . "tr0" : jump servers_to_internet\
}\
\# Everything else (e.g. DMZ -\> internal zones, Outside -\> internal zones) is denied and logged\
log prefix "CORE-FWD-DROP: " counter drop\
}\
\# ---------- Layer 7: block web-admin pages and unsafe HTTP methods from the Outside ----------\
\# @ih = start of the TCP payload, i.e. the HTTP request line\
chain web_inspect {\
@ih,0,80 0x474554202f61646d696e log prefix "CORE-L7-ADMIN-BLOCK: " counter drop \# "GET /admin" (Joomla /administrator, PrestaShop /admin...)\
@ih,0,88 0x504f5354202f61646d696e log prefix "CORE-L7-ADMIN-BLOCK: " counter drop \# "POST /admin"\
@ih,0,40 0x5452414345 log prefix "CORE-L7-METHOD-BLOCK: " counter drop \# "TRACE"\
}\
\# ---------- zone-pair policies (NEW connections only; replies use the fast path) ----------\
chain staff_to_dmz { \# Users/Admin -\> DMZ\
ip daddr @pubweb tcp dport @web_ports accept\
ip daddr \$PUBDNS meta l4proto { tcp, udp } th dport 53 accept\
ip saddr @admins ip daddr @linux_srv tcp dport 22 accept\
icmp type echo-request accept\
}\
chain staff_to_internal { \# Users/Admin -\> Internal apps\
ip daddr \$INTWEB tcp dport @web_ports accept\
ip daddr \$FILESRV tcp dport 445 accept\
ip saddr @admins ip daddr @linux_srv tcp dport 22 accept\
icmp type echo-request accept
