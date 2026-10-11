# Contents

[1. Network design diagram](#network-design-diagram)

[2. Business requirements](#business-requirements)

[3. Network devices and addressing](#network-devices-and-addressing)

[4. Software](#software)

[5. Security zones](#security-zones)

[6. Security zone boundaries](#security-zone-boundaries)

# 1. Network design diagram

![](images/migration-plan/media/image6.png)

*Figure 3 – Group 6 target topology: Edge router, Core router and three Open vSwitch switches joined by 802.1Q trunks; one VLAN per security zone*

# 2. Business requirements

| **Req_ID** | **Requirement description** | **Test case description** | **Acceptance criteria** |
|----|----|----|----|
| BR-01 | Separate the network into security zones – Users (VLAN 10), Admin (VLAN 20), DMZ (VLAN 30), Internal (VLAN 40), Infra (VLAN 50), Transit and Outside – each with its own address block derived from x = 6. | On the Core run 'ip -br a'; on SW1–SW3 run 'ovs-vsctl show'; check every host is on an access port of its zone. | Core: vl10 192.168.6.1/25, vl20 192.168.6.129/28, vl30 172.16.6.1/25, vl40 172.16.6.129/25, vl50 10.6.6.1/24, tr0 192.168.6.146/30. Every host is in exactly one VLAN. |
| BR-02 | Implement the zones as real 802.1Q VLANs on virtual switches (Open vSwitch VMs) instead of physical switches. | Run 'ovs-vsctl show' on SW1–SW3 and 'tcpdump -e -n -i uplink vlan' on SW2 while a user browses; run 'ip -d link show vl10' on the Core. | Access ports carry one VLAN tag; trunks carry only the VLANs listed (SW1 10,20 · SW2 10–50 · SW3 40,50); SW2 has DMZ access ports only with no native VLAN; captured frames show 802.1Q tags; Core sub-interfaces are 'vlan protocol 802.1Q'. |
| BR-03 | Replace static routing with OSPF (FRR) between the Edge and the Core. | On Edge and Core: 'vtysh -c "show ip ospf neighbor"' and 'show ip route ospf'. | The neighbour is Full; the Core learns 0.0.0.0/0 and 10.10.10.0/24 as 'O' routes and the Edge learns every internal VLAN; no static routes. |
| BR-04 | Protect the routing protocol against rogue routers. | 'show ip ospf interface tr0' on both routers; connect a router without the key. | MD5 authentication is shown on tr0; a router without the key never becomes a neighbour. All VLAN interfaces are passive. |
| BR-05 | Public services (CorpWeb – Joomla, E-Site – PrestaShop) are reachable from the Outside on HTTP/HTTPS only. | From the Outside User PC browse http://corpweb.group6.local and http://esite.group6.local; from Kali run 'nmap -Pn 172.16.6.11 172.16.6.14'. | Both sites load; nmap shows only 80/443 open, every other port filtered. |
| BR-06 | Internal applications (IntWeb – Odoo, FileSharing – Samba) and all internal zones are not reachable from the Outside. | From the Outside User PC browse http://172.16.6.132 and open \\\172.16.6.133; from Kali run 'nmap -Pn -p 22,80,389,445,3389 172.16.6.132 172.16.6.133 10.6.6.10 192.168.6.130'. | Browser and Explorer time out; nmap reports everything filtered; drops are logged (EDGE-FWD-DROP / CORE-FWD-DROP). |
| BR-07 | Employees can use all corporate applications (Joomla, Odoo, PrestaShop) and the file share. | On the User PC open http://corpweb.group6.local, http://intweb.group6.local and http://esite.group6.local, and \\\172.16.6.133\share in Explorer. | Joomla site, Odoo login page and PrestaShop shop load; the share opens with the g6user account. |
| BR-08 | Private DNS answers only authorized internal endpoints (Users, Admin, Internal, Infra). | Query 10.6.6.12 from the User PC, from the Outside User PC and from the DMZ. | The User PC gets answers (internal and Internet names); queries from the Outside and from the DMZ time out. |
| BR-09 | Public DNS (172.16.6.53, DMZ) answers any endpoint, exposes only public records and does not recurse. | From the Outside User PC run nslookup for corpweb, esite, intweb and ubuntu.com. | corpweb → 172.16.6.11, esite → 172.16.6.14; intweb returns Non-existent domain; ubuntu.com is refused. |
| BR-10 | Clients resolve names only through the private DNS; AD is a separate server from DNS. | On the User PC run 'nslookup intweb.group6.local', 'nltest /dsgetdc:group6.local' and 'nslookup group6.local 10.6.6.10'. | Names and the domain controller are found through 10.6.6.12; the direct query to AD times out. |
| BR-11 | Workstations are members of group6.local and users log on with domain accounts. | Log on to the User PC as GROUP6\employee1 and to the Admin PC as GROUP6\itadmin; run 'whoami'. | Logon succeeds; whoami shows the domain account. |
| BR-12 | The Users VLAN receives addresses from the central DHCP server in the Infra VLAN. | Start the User PC with DHCP; run 'ipconfig /all'; check isc-dhcp-relay on the Core. | The User PC receives 192.168.6.50–.120, gateway 192.168.6.1 and DNS 10.6.6.12 through the Core relay. |
| BR-13 | Routers and servers can be managed only from the Admin PC (192.168.6.130). | From the Admin PC: 'ssh ubuntu@192.168.6.129' (Core), 'ssh sysadmin@192.168.6.145' (Edge), 'mstsc /v:10.6.6.10'. Repeat the port tests from the User PC. | SSH and RDP work from the Admin PC only; from the User PC every management connection fails. |
| BR-14 | The Admin PC is isolated in its own VLAN: a compromised user PC can neither reach it nor impersonate it. | From the User PC run 'Test-NetConnection 192.168.6.130 -Port 3389'; then set the User PC to 192.168.6.130 and try SSH to the Core and RDP to AD; check the Core journal. | Every attempt fails; the Core logs CORE-USR-ADMIN-DROP and CORE-SPOOF-DROP (Admin address seen on VLAN 10). |
| BR-15 | A compromised DMZ server cannot reach the Users, Admin, Internal or Infra VLANs. | From the WebPublic host connect to 172.16.6.132:80, 172.16.6.133:445, 10.6.6.10:389 and query 10.6.6.12. | All attempts are blocked and logged (CORE-FWD-DROP). |
| BR-16 | Internal zones have Internet access through the Edge (NAT). | From the User PC open https://ubuntu.com; on the Linux servers run 'apt update'. | Connections succeed and leave with the Edge WAN address. |
| BR-17 | Zone boundaries use stateless firewalls with default deny, anti-spoofing and logging on the Edge and the Core. | Run 'nft list chain inet filter forward' on Edge and Core; send a disallowed packet and check the journal. | Forward chains have policy drop and no 'ct state' matches; drops are logged with the router prefix. |
| BR-18 | Application databases (MariaDB, PostgreSQL) are never reachable from any VLAN. | From the User PC run 'Test-NetConnection 172.16.6.11 -Port 3306' and '... 172.16.6.132 -Port 5432'; from Kali scan 3306/5432. | No database port is reachable; databases sit only on internal Docker networks (be_joomla, be_presta, be_odoo). |
| BR-19 | Applications and administrator accounts do not use default credentials. | Try admin/admin on Joomla (/administrator), Odoo and PrestaShop (/admin-g6). | Default credentials are rejected; each application uses a unique strong password. |
| BR-20 | Public and internal applications run on separate servers in separate VLANs. | Run 'docker ps' on WebPublic (VLAN 30) and AppsInternal (VLAN 40). | WebPublic runs only Joomla, PrestaShop and MariaDB; AppsInternal runs only Odoo, PostgreSQL and Samba. |

*Table 1 – List of business requirements*

# 3. Network devices and addressing

<table>
<colgroup>
<col style="width: 16%" />
<col style="width: 18%" />
<col style="width: 8%" />
<col style="width: 14%" />
<col style="width: 14%" />
<col style="width: 10%" />
<col style="width: 12%" />
<col style="width: 6%" />
</colgroup>
<thead>
<tr>
<th><p><strong>FQDN</strong></p></th>
<th><p><strong>Role</strong></p></th>
<th><p><strong>Interface</strong></p></th>
<th><p><strong>Old IP address (CIDR)</strong></p></th>
<th><p><strong>New IP address (CIDR)</strong></p></th>
<th><p><strong>Old gateway</strong></p></th>
<th><p><strong>New gateway</strong></p></th>
<th><p><strong>VLAN</strong></p></th>
</tr>
</thead>
<tbody>
<tr>
<td><p>edge.group6.local</p></td>
<td><p>Edge router – perimeter firewall, NAT, OSPF</p></td>
<td><p>wan0</p>
<p>out0</p>
<p>tr0</p></td>
<td><p>DHCP (NAT)</p>
<p>10.10.10.1/24</p>
<p>192.168.6.145/30</p></td>
<td><p>DHCP (NAT)</p>
<p>10.10.10.1/24</p>
<p>192.168.6.145/30</p></td>
<td><p>VMware NAT</p></td>
<td><p>VMware NAT</p></td>
<td><p>—</p>
<p>Outside</p>
<p>Transit</p></td>
</tr>
<tr>
<td><p>core.group6.local</p></td>
<td><p>Core router – inter-VLAN routing, zone firewall, DHCP relay, OSPF</p></td>
<td><p>tr0</p>
<p>vl10</p>
<p>vl20</p>
<p>vl30</p>
<p>vl40</p>
<p>vl50</p></td>
<td><p>192.168.6.146/30</p>
<p>192.168.6.129/28 (LAN)</p>
<p>—</p>
<p>—</p>
<p>—</p>
<p>—</p></td>
<td><p>192.168.6.146/30</p>
<p>192.168.6.1/25</p>
<p>192.168.6.129/28</p>
<p>172.16.6.1/25</p>
<p>172.16.6.129/25</p>
<p>10.6.6.1/24</p></td>
<td><p>192.168.6.145 (static)</p></td>
<td><p>192.168.6.145 (OSPF)</p></td>
<td><p>Transit</p>
<p>10</p>
<p>20</p>
<p>30</p>
<p>40</p>
<p>50</p></td>
</tr>
<tr>
<td><p>sw1 (new)</p></td>
<td><p>Open vSwitch – Users/Admin access switch</p></td>
<td><p>uplink</p>
<p>p-user</p>
<p>p-admin</p></td>
<td><p>—</p></td>
<td><p>No IP (layer 2)</p></td>
<td><p>—</p></td>
<td><p>—</p></td>
<td><p>trunk 10,20</p>
<p>10</p>
<p>20</p></td>
</tr>
<tr>
<td><p>sw2 (new)</p></td>
<td><p>Open vSwitch – distribution + DMZ switch</p></td>
<td><p>uplink</p>
<p>to-sw1 / to-sw3</p>
<p>p-webpublic, p-publicdns</p></td>
<td><p>—</p></td>
<td><p>No IP (layer 2)</p></td>
<td><p>—</p></td>
<td><p>—</p></td>
<td><p>trunk 10–50</p>
<p>trunk 10,20 / 40,50</p>
<p>30</p></td>
</tr>
<tr>
<td><p>sw3 (new)</p></td>
<td><p>Open vSwitch – internal + infrastructure switch</p></td>
<td><p>uplink</p>
<p>p-appsint</p>
<p>p-ad, p-dhcp, p-privdns</p></td>
<td><p>—</p></td>
<td><p>No IP (layer 2)</p></td>
<td><p>—</p></td>
<td><p>—</p></td>
<td><p>trunk 40,50</p>
<p>40</p>
<p>50</p></td>
</tr>
<tr>
<td><p>admin.group6.local</p></td>
<td><p>Admin workstation – Windows 10, domain member</p></td>
<td><p>Ethernet0</p></td>
<td><p>192.168.6.130/28</p></td>
<td><p>192.168.6.130/28 (static)</p></td>
<td><p>192.168.6.129</p></td>
<td><p>192.168.6.129 (Core vl20)</p></td>
<td><p>20</p></td>
</tr>
<tr>
<td><p>user.group6.local</p></td>
<td><p>Employee workstation – Windows 10, domain member</p></td>
<td><p>Ethernet0</p></td>
<td><p>192.168.6.131/28</p></td>
<td><p>DHCP 192.168.6.50–.120</p></td>
<td><p>192.168.6.129</p></td>
<td><p>192.168.6.1 (Core vl10)</p></td>
<td><p>10</p></td>
</tr>
<tr>
<td><p>webpublic.group6.local</p></td>
<td><p>Docker host – public web applications (new)</p></td>
<td><p>dmz0</p></td>
<td><p>— (new)</p></td>
<td><p>172.16.6.10/25</p></td>
<td><p>—</p></td>
<td><p>172.16.6.1</p></td>
<td><p>30</p></td>
</tr>
<tr>
<td><p>corpweb.group6.local</p></td>
<td><p>Corporate website – Joomla (container)</p></td>
<td><p>eth0 (macvlan)</p></td>
<td><p>192.168.6.132/28</p></td>
<td><p>172.16.6.11/25</p></td>
<td><p>192.168.6.129</p></td>
<td><p>172.16.6.1</p></td>
<td><p>30</p></td>
</tr>
<tr>
<td><p>esite.group6.local</p></td>
<td><p>E-commerce – PrestaShop (container)</p></td>
<td><p>eth0 (macvlan)</p></td>
<td><p>192.168.6.132/28 (shared)</p></td>
<td><p>172.16.6.14/25</p></td>
<td><p>192.168.6.129</p></td>
<td><p>172.16.6.1</p></td>
<td><p>30</p></td>
</tr>
<tr>
<td><p>pubdns.group6.local</p></td>
<td><p>Public authoritative DNS (new)</p></td>
<td><p>dmz0</p></td>
<td><p>— (new)</p></td>
<td><p>172.16.6.53/25</p></td>
<td><p>—</p></td>
<td><p>172.16.6.1</p></td>
<td><p>30</p></td>
</tr>
<tr>
<td><p>db (container)</p></td>
<td><p>MariaDB for Joomla / PrestaShop</p></td>
<td><p>internal bridges</p></td>
<td><p>— (new)</p></td>
<td><p>Docker networks only</p></td>
<td><p>—</p></td>
<td><p>— (not routed)</p></td>
<td><p>30 (int.)</p></td>
</tr>
<tr>
<td><p>appsinternal.group6.local</p></td>
<td><p>Docker host – internal applications (new)</p></td>
<td><p>app0</p></td>
<td><p>— (new)</p></td>
<td><p>172.16.6.130/25</p></td>
<td><p>—</p></td>
<td><p>172.16.6.129</p></td>
<td><p>40</p></td>
</tr>
<tr>
<td><p>intweb.group6.local</p></td>
<td><p>Internal web application – Odoo (container)</p></td>
<td><p>eth0 (macvlan)</p></td>
<td><p>192.168.6.133/28</p></td>
<td><p>172.16.6.132/25</p></td>
<td><p>192.168.6.129</p></td>
<td><p>172.16.6.129</p></td>
<td><p>40</p></td>
</tr>
<tr>
<td><p>files.group6.local</p></td>
<td><p>File sharing – Samba (container)</p></td>
<td><p>eth0 (macvlan)</p></td>
<td><p>192.168.6.134/28</p></td>
<td><p>172.16.6.133/25</p></td>
<td><p>192.168.6.129</p></td>
<td><p>172.16.6.129</p></td>
<td><p>40</p></td>
</tr>
<tr>
<td><p>odoodb (container)</p></td>
<td><p>PostgreSQL for Odoo</p></td>
<td><p>internal bridge</p></td>
<td><p>— (new)</p></td>
<td><p>Docker network only</p></td>
<td><p>—</p></td>
<td><p>— (not routed)</p></td>
<td><p>40 (int.)</p></td>
</tr>
<tr>
<td><p>dc.group6.local (ad)</p></td>
<td><p>AD domain controller (AD DS + AD DNS)</p></td>
<td><p>Ethernet0</p></td>
<td><p>192.168.6.135/28</p></td>
<td><p>10.6.6.10/24</p></td>
<td><p>192.168.6.129</p></td>
<td><p>10.6.6.1</p></td>
<td><p>50</p></td>
</tr>
<tr>
<td><p>dhcp.group6.local (netsvc)</p></td>
<td><p>DHCP server</p></td>
<td><p>svc0</p></td>
<td><p>192.168.6.134/28</p></td>
<td><p>10.6.6.11/24</p></td>
<td><p>192.168.6.129</p></td>
<td><p>10.6.6.1</p></td>
<td><p>50</p></td>
</tr>
<tr>
<td><p>privdns.group6.local</p></td>
<td><p>Private DNS resolver (new)</p></td>
<td><p>svc0</p></td>
<td><p>— (new)</p></td>
<td><p>10.6.6.12/24</p></td>
<td><p>—</p></td>
<td><p>10.6.6.1</p></td>
<td><p>50</p></td>
</tr>
<tr>
<td><p>OUTSIDE (workgroup)</p></td>
<td><p>Outside / Internet user – Windows 10</p></td>
<td><p>Ethernet0</p></td>
<td><p>10.10.10.2/24</p></td>
<td><p>10.10.10.2/24 (DNS 172.16.6.53)</p></td>
<td><p>10.10.10.1</p></td>
<td><p>10.10.10.1</p></td>
<td><p>Outside</p></td>
</tr>
<tr>
<td><p>attacker</p></td>
<td><p>Attacker / vulnerability scanner – Kali Linux</p></td>
<td><p>eth0</p></td>
<td><p>10.10.10.3/24</p></td>
<td><p>10.10.10.3/24 (DNS 172.16.6.53)</p></td>
<td><p>10.10.10.1</p></td>
<td><p>10.10.10.1</p></td>
<td><p>Outside</p></td>
</tr>
</tbody>
</table>

*Table 2 – List of network devices*

# 4. Software

<table>
<colgroup>
<col style="width: 19%" />
<col style="width: 19%" />
<col style="width: 20%" />
<col style="width: 23%" />
<col style="width: 15%" />
</colgroup>
<thead>
<tr>
<th><p><strong>FQDN</strong></p></th>
<th><p><strong>Role</strong></p></th>
<th><p><strong>OS</strong></p></th>
<th><p><strong>Software</strong></p></th>
<th><p><strong>Version</strong></p></th>
</tr>
</thead>
<tbody>
<tr>
<td><p>edge.group6.local</p></td>
<td><p>Edge router, perimeter firewall, NAT</p></td>
<td><p>Ubuntu Server 24.04.1 LTS</p></td>
<td><p>FRRouting (ospfd)</p>
<p>nftables</p>
<p>OpenSSH server</p></td>
<td><p>8.4.4</p>
<p>1.0.9</p>
<p>9.6p1</p></td>
</tr>
<tr>
<td><p>core.group6.local</p></td>
<td><p>Core router, inter-VLAN firewall, DHCP relay</p></td>
<td><p>Ubuntu Server 24.04.5 LTS</p></td>
<td><p>FRRouting (ospfd)</p>
<p>nftables</p>
<p>isc-dhcp-relay</p>
<p>802.1Q sub-interfaces (netplan / systemd-networkd)</p>
<p>OpenSSH server</p></td>
<td><p>8.4.4</p>
<p>1.0.9</p>
<p>4.4.3-P1</p>
<p>Ubuntu 24.04</p>
<p>9.6p1</p></td>
</tr>
<tr>
<td><p>sw1 / sw2 / sw3</p></td>
<td><p>Virtual switches (802.1Q trunks and access ports)</p></td>
<td><p>Ubuntu Server 24.04.1 LTS</p></td>
<td><p>Open vSwitch</p></td>
<td><p>3.3.9</p></td>
</tr>
<tr>
<td><p>dc.group6.local</p></td>
<td><p>Domain controller</p></td>
<td><p>Windows Server 2019 (build 17763)</p></td>
<td><p>Active Directory Domain Services</p>
<p>DNS Server (AD-integrated)</p>
<p>Remote Desktop (Admin PC only)</p></td>
<td><p>Windows Server 2019</p>
<p>Windows Server 2019</p>
<p>Built-in</p></td>
</tr>
<tr>
<td><p>dhcp.group6.local</p></td>
<td><p>DHCP server</p></td>
<td><p>Ubuntu Server 24.04.5 LTS</p></td>
<td><p>isc-dhcp-server</p></td>
<td><p>4.4.3-P1</p></td>
</tr>
<tr>
<td><p>privdns / pubdns</p></td>
<td><p>Private resolver / public authoritative DNS</p></td>
<td><p>Ubuntu Server 24.04.1 LTS</p></td>
<td><p>BIND9 (named)</p>
<p>nftables</p></td>
<td><p>9.18.39</p>
<p>1.0.9</p></td>
</tr>
<tr>
<td><p>webpublic / appsinternal</p></td>
<td><p>Container hosts (DMZ / Internal)</p></td>
<td><p>Ubuntu Server 24.04.1 LTS</p></td>
<td><p>Docker Engine</p>
<p>Docker Compose v2</p>
<p>nftables</p></td>
<td><p>29.1.3</p>
<p>Ubuntu package</p>
<p>1.0.9</p></td>
</tr>
<tr>
<td><p>corpweb.group6.local</p></td>
<td><p>Corporate website (CMS)</p></td>
<td><p>Debian (joomla:5-apache)</p></td>
<td><p>Joomla</p>
<p>Apache HTTP Server</p>
<p>PHP</p></td>
<td><p>5.4.9</p>
<p>2.4.68</p>
<p>8.3.35</p></td>
</tr>
<tr>
<td><p>esite.group6.local</p></td>
<td><p>E-commerce shop</p></td>
<td><p>Debian (prestashop/prestashop:8)</p></td>
<td><p>PrestaShop</p>
<p>Apache HTTP Server</p></td>
<td><p>8.2.8</p>
<p>2.4.65</p></td>
</tr>
<tr>
<td><p>intweb.group6.local</p></td>
<td><p>Internal collaboration / ERP</p></td>
<td><p>Debian (odoo:17)</p></td>
<td><p>Odoo (built-in web server, port 80)</p></td>
<td><p>17.0</p></td>
</tr>
<tr>
<td><p>files.group6.local</p></td>
<td><p>File sharing</p></td>
<td><p>Alpine Linux (dperson/samba)</p></td>
<td><p>Samba (smbd)</p></td>
<td><p>4.12.2</p></td>
</tr>
<tr>
<td><p>db / odoodb (containers)</p></td>
<td><p>Application databases</p></td>
<td><p>Debian (mariadb:11, postgres:16)</p></td>
<td><p>MariaDB</p>
<p>PostgreSQL</p></td>
<td><p>11.x</p>
<p>16.x</p></td>
</tr>
<tr>
<td><p>admin / user / OUTSIDE</p></td>
<td><p>Workstations</p></td>
<td><p>Windows 10 Pro (build 19041)</p></td>
<td><p>Microsoft Edge, OpenSSH client, Remote Desktop client</p>
<p>VMware Tools</p></td>
<td><p>Built-in</p>
<p>12.4.5</p></td>
</tr>
<tr>
<td><p>attacker</p></td>
<td><p>Vulnerability scanning</p></td>
<td><p>Kali Linux 2025.4</p></td>
<td><p>Nmap, Nikto, OWASP ZAP, Firefox ESR</p></td>
<td><p>Kali 2025.4 packages</p></td>
</tr>
<tr>
<td><p>Host PC</p></td>
<td><p>Virtualization platform</p></td>
<td><p>Windows 11</p></td>
<td><p>VMware Workstation Pro (LAN segments used as virtual cables)</p></td>
<td><p>17.6.0</p></td>
</tr>
</tbody>
</table>

*Table 3 – List of software*

# 5. Security zones

<table>
<colgroup>
<col style="width: 20%" />
<col style="width: 47%" />
<col style="width: 31%" />
</colgroup>
<thead>
<tr>
<th><p><strong>Security zone name</strong></p></th>
<th><p><strong>Description</strong></p></th>
<th><p><strong>Members (network devices)</strong></p></th>
</tr>
</thead>
<tbody>
<tr>
<td><p>Outside (untrusted)</p>
<p>10.10.10.0/24</p></td>
<td><p>Internet and external users. No trust; may only reach the published services (CorpWeb, E-Site, public DNS).</p></td>
<td><p>Outside User PC 10.10.10.2, Attacker (Kali) 10.10.10.3, Internet (via Edge NAT)</p></td>
</tr>
<tr>
<td><p>Transit</p>
<p>192.168.6.144/30</p></td>
<td><p>Direct routed link between Edge and Core. Carries OSPF (MD5) and routed traffic only; never passes through a switch VM.</p></td>
<td><p>edge tr0 .145, core tr0 .146</p></td>
</tr>
<tr>
<td><p>Users – VLAN 10</p>
<p>192.168.6.0/25 (SW1)</p></td>
<td><p>Employee workstations. May start connections to the applications, Infra services and the Internet; no zone may start a connection into it and it cannot reach the Admin VLAN.</p></td>
<td><p>user (Windows 10, DHCP)</p></td>
</tr>
<tr>
<td><p>Admin – VLAN 20</p>
<p>192.168.6.128/28 (SW1)</p></td>
<td><p>Privileged administration workstation, separated from ordinary users. The only source allowed to manage routers and servers; the Core accepts the Admin address only on VLAN 20.</p></td>
<td><p>admin (Windows 10) 192.168.6.130</p></td>
</tr>
<tr>
<td><p>DMZ – VLAN 30</p>
<p>172.16.6.0/25 (SW2)</p></td>
<td><p>Internet-facing servers. Assumed to be attacked: may not start any connection to Users, Admin, Internal or Infra and does not use the private DNS. SW2 has DMZ access ports only.</p></td>
<td><p>webpublic .10, corpweb/Joomla .11, esite/PrestaShop .14, pubdns .53; MariaDB on internal Docker networks</p></td>
</tr>
<tr>
<td><p>Internal – VLAN 40</p>
<p>172.16.6.128/25 (SW3)</p></td>
<td><p>Applications used only by employees. Reachable from Users and Admin only; may only use the private DNS.</p></td>
<td><p>appsinternal .130, intweb/Odoo .132, files/Samba .133; PostgreSQL on an internal Docker network</p></td>
</tr>
<tr>
<td><p>Infra – VLAN 50</p>
<p>10.6.6.0/24 (SW3)</p></td>
<td><p>Identity, addressing and name resolution. Most restricted: only required service ports reachable, management only from the Admin PC. Shares SW3 only with the Internal VLAN, never with the DMZ.</p></td>
<td><p>ad (DC) 10.6.6.10, netsvc (DHCP) 10.6.6.11, privdns 10.6.6.12</p></td>
</tr>
</tbody>
</table>

*Table 4 – List of security zones*

# 6. Security zone boundaries

| **From security zone** | **To security zone** | **Protocol** | **Reason** |
|----|----|----|----|
| Outside | DMZ (CorpWeb .11, E-Site .14) | TCP 80, 443 | Public website and online store. (Edge, Core) |
| Outside | DMZ (pubdns .53) | UDP/TCP 53 | Anyone can resolve the public names. (Edge, Core) |
| Outside | Users, Admin, Internal, Infra | Any – DENY | Internal resources are never exposed. (Edge, Core) |
| Outside / Internet | Any (source address from an internal block) | Any – DENY + log | Anti-spoofing on the Edge (EDGE-SPOOF-DROP). |
| Any VLAN | Any (source address outside its own subnet) | Any – DENY + log | Anti-spoofing on every Core sub-interface (CORE-SPOOF-DROP). |
| DMZ | Users, Admin, Internal, Infra | Any – DENY | A compromised public server cannot pivot inward. (Core) |
| DMZ (webpublic, pubdns) | Internet | TCP 53, 80, 443; UDP 53 | Updates and image pulls via public resolvers. (Core, Edge) |
| Users, Admin | DMZ (CorpWeb, E-Site, pubdns) | TCP 80, 443; UDP/TCP 53 | Employees use the public sites. (Core) |
| Users, Admin | Internal (intweb .132) | TCP 80, 443 | Employees use Odoo. (Core) |
| Users, Admin | Internal (files .133) | TCP 445 (SMB) | Employee file sharing. (Core) |
| Users, Admin | Infra (privdns .12) | UDP/TCP 53 | Name resolution. (Core) |
| Users, Admin | Infra (ad .10) | TCP 88, 135, 389, 445, 464, 636, 3268–3269, 49152–65535; UDP 88, 123, 389, 464 | Domain join and logon, Group Policy, time sync. (Core) |
| Users, Admin | Infra (ad .10) | UDP/TCP 53 – DENY | Clients must use the private DNS. (Core) |
| Users (DHCP clients) | Core relay → Infra (netsvc .11) | UDP 67/68 | Central address assignment. (Core) |
| Admin (192.168.6.130 only) | Edge, Core, DMZ/Internal/Infra servers | TCP 22 (SSH); TCP 3389 (RDP to AD) | Administration from one management host. (Core, Edge, host firewalls) |
| Admin | Users | ICMP echo; TCP 3389 | Remote support of user PCs. (Core) |
| Users | Admin | Any – DENY + log | A compromised user PC cannot reach the Admin PC. (CORE-USR-ADMIN-DROP) |
| Users, Admin | Internet / Outside | Any (replies: TCP without SYN) | Browsing and updates. (Core, Edge) |
| Any zone | Users, Admin (new connections) | Any – DENY | Workstations are clients only. (Core) |
| Internal | Infra (privdns .12) | UDP/TCP 53 | Application servers resolve names. (Core) |
| Internal | Users, Admin, DMZ, Infra (other) | Any – DENY | Application servers only answer requests. (Core) |
| Internal (appsinternal host) | Internet | TCP 80, 443 | Image pulls and updates. (Core, Edge) |
| Infra (privdns, ad, netsvc) | Internet | UDP/TCP 53, UDP 123, TCP 80/443 | DNS forwarding, time, updates. (Core, Edge) |
| Application containers | Their own database | TCP 3306 / 5432 on internal Docker networks | One private network per application; not routed. |
| Any zone | Databases | Any – DENY (no route, no VLAN IP) | Databases are never exposed. |
| Edge | Core (Transit) | OSPF (IP protocol 89), MD5 | Dynamic routing. |
| Switch trunks | — | Only the listed VLANs; no native VLAN | Limits VLAN hopping: SW1 10,20 · SW2 10–50 · SW3 40,50; the DMZ switch has no internal access ports. |
| Any | Any (not listed above) | Any – DENY + log | Default-deny forward policy on Edge and Core. |

*Table 5 – Security zone boundaries*
