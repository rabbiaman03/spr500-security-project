<div align="center">

# 🏢 SPR500 — Security Policies and Network Security

**Designing, building, segmenting and assessing a virtual enterprise network**

![VMware](https://img.shields.io/badge/VMware-Workstation-607078?logo=vmware&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-nftables-FCC624?logo=linux&logoColor=black)
![Windows Server](https://img.shields.io/badge/Windows_Server-AD_DS-0078D6?logo=windows&logoColor=white)
![Open vSwitch](https://img.shields.io/badge/Open_vSwitch-802.1Q-blue)

</div>

---

## 📌 Overview

Group project from **SPR500 (Fall 2026, Seneca Polytechnic)**, completed as a member of **Group 6**, together with teammate [@cybersaviour-AS](https://github.com/cybersaviour-AS). We built the enterprise network `group6.local` in VMware, hardened it from a flat design into a zoned one, and assessed its risks.

![Lab 1 topology](lab01/topology.png)

## 🔬 Stages

| Stage | Title | What was built | Report |
|---|---|---|---|
| Lab 01 | **Enterprise network design and implementation** | Edge router with nftables, Active Directory, DNS and DHCP, web apps (Joomla, PrestaShop, Odoo) and Samba file sharing. Business requirements mapped to test cases | [Report](lab01/README.md) · [TVRA](lab01/tvra-report.md) |
| Lab 02 | **Security zones** | Migration to 5 VLAN zones (Users, Admin, DMZ, Internal, Infra) on Open vSwitch, OSPF with MD5 authentication (FRR), stateless default-deny nftables with anti-spoofing, split private/public DNS, DMZ isolation, vulnerability scans | [Report](lab02/README.md) · [Migration plan](lab02/migration-plan.md) |
| Lab 03 | **Network segregation policies** | Stateful Core firewall (conntrack, verdict maps, Layer-7 `/admin` blocking, SYN-flood meter) and default-deny host firewalls on every machine, deployed by **Ansible** | [Overview](lab03/README.md) · [Migration plan](lab03/migration-plan.md) |
| Project | **Stage 1: OSINT technical specification** | Passive-only reconnaissance of a public company's attack surface, with defined rules of engagement | [Report](project/README.md) |

## 🛡️ Security controls implemented

| Control | Implementation |
|---|---|
| Network segmentation | 802.1Q VLANs per security zone, Core router sub-interfaces |
| Perimeter filtering | nftables default deny, anti-spoofing, logging |
| Stateful inspection | Core firewall tracks connections, drops invalid packets, requires SYN for new TCP flows; Layer-7 blocking of admin paths; SYN-flood rate limiting |
| Host-based firewalls | Default-deny on every server and PC, deployed and verified with Ansible |
| Routing security | OSPF area 0 with MD5 authentication against rogue routers |
| DNS security | Private resolver for internal clients only; public DNS with no recursion |
| Least-privilege management | Routers and servers manageable only from an isolated Admin VLAN |
| Risk management | Threat and vulnerability risk assessment (TVRA) with likelihood, impact and mitigations |

## 👥 Team

| Member | GitHub |
|---|---|
| Rabbi Muhammad Aman | [@rabbiaman03](https://github.com/rabbiaman03) |
| Anjali Solanki | [@cybersaviour-AS](https://github.com/cybersaviour-AS) |

All labs (01–03) and the project were completed as Group 6.

## 🧰 Tools and skills

`VMware` · `Linux` · `nftables` · `FRRouting (OSPF)` · `Open vSwitch` · `Windows Server / AD DS` · `DNS / DHCP` · `Docker` · `Kali Linux` · `Ansible` · `Vulnerability scanning` · `Risk assessment` · `OSINT`

---

<sub>Author: <b>Rabbi Muhammad Aman</b> (Group 6) · Academic group coursework built in an isolated lab environment. The OSINT stage used passive public sources only.</sub>
