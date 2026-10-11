SPR500 NBB — Final Project

**Stage 1: Project Conceptualization**

OSINT Technical Specification — DAVIDsTEA Inc.

| **Group**           | Group 6                                             |
|---------------------|-----------------------------------------------------|
| **Student**         | Rabbi Muhammad Aman                     |
| **Course**          | SPR500 NBB — Security Policies and Network Security |
| **Professor**       | \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_            |
| **Submission date** | \_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_\_            |

*All information in this document was gathered by passive open-source intelligence only. No network traffic was sent to any DAVIDsTEA asset, in line with the project rules and Canadian law.*

**Table of Contents**

[1. Executive Summary [3](#executive-summary)](#executive-summary)

[2. Company Selection [3](#company-selection)](#company-selection)

[2.1 Selection criteria [3](#selection-criteria)](#selection-criteria)

[2.2 Alternatives considered [4](#alternatives-considered)](#alternatives-considered)

[3. Company Profile [4](#company-profile)](#company-profile)

[3.1 Business functions that depend on IT [4](#business-functions-that-depend-on-it)](#business-functions-that-depend-on-it)

[4. OSINT Methodology and Rules of Engagement [5](#osint-methodology-and-rules-of-engagement)](#osint-methodology-and-rules-of-engagement)

[4.1 Rules of engagement [5](#rules-of-engagement)](#rules-of-engagement)

[4.2 Sources used [5](#sources-used)](#sources-used)

[4.3 Further passive sources for later stages [5](#further-passive-sources-for-later-stages)](#further-passive-sources-for-later-stages)

[5. Public-Facing Services [5](#public-facing-services)](#public-facing-services)

[6. Internal IT Infrastructure Services [6](#internal-it-infrastructure-services)](#internal-it-infrastructure-services)

[7. Technical Specifications [7](#technical-specifications)](#technical-specifications)

[7.1 Known and probable technology stack [7](#known-and-probable-technology-stack)](#known-and-probable-technology-stack)

[7.2 Estimated asset inventory [8](#estimated-asset-inventory)](#estimated-asset-inventory)

[8. Initial Threat Landscape [8](#initial-threat-landscape)](#initial-threat-landscape)

[9. Laws, Regulations and Pending Legislation [8](#laws-regulations-and-pending-legislation)](#laws-regulations-and-pending-legislation)

[9.1 In force [8](#in-force)](#in-force)

[9.2 Pending or proposed legislation [9](#pending-or-proposed-legislation)](#pending-or-proposed-legislation)

[9.3 Frameworks we will use [9](#frameworks-we-will-use)](#frameworks-we-will-use)

[10. Proposed Scope for the Security Architecture [10](#proposed-scope-for-the-security-architecture)](#proposed-scope-for-the-security-architecture)

[10.1 Mapping DAVIDsTEA to our lab environment [10](#mapping-davidstea-to-our-lab-environment)](#mapping-davidstea-to-our-lab-environment)

[10.2 Preliminary security requirements [10](#preliminary-security-requirements)](#preliminary-security-requirements)

[10.3 Assumptions and limitations [10](#assumptions-and-limitations)](#assumptions-and-limitations)

[11. References [10](#references)](#references)

# 1. Executive Summary

Group 6 has selected DAVIDsTEA Inc., a Montréal-based specialty tea retailer, as the subject of the SPR500 final project. DAVIDsTEA is a good fit for the brief: it is a Canadian mid-sized business (about 250 full-time staff and \$61.0M in sales in fiscal 2025), it handles payment cards and the personal information of hundreds of thousands of loyalty members, and it runs a mix of public-facing and internal IT services across a head office, a production facility, 21 retail stores, warehouses and third-party logistics partners.

Using only passive OSINT (public filings, press releases, job postings, agency case studies and other third-party sources), we identified the company's main public-facing services, the internal IT services it needs to operate, and the technology it is known or likely to use. We also identified the Canadian (and, because of its U.S. online sales, some American) laws and pending legislation that will shape the security design.

**Key findings:**

- **Online is the largest channel.** E-commerce produced 46.9% of fiscal 2025 sales, so the web store is the most business-critical asset. The store was migrated from Salesforce Commerce Cloud to Shopify Plus, with Shopify POS in stores.

- **Third-party heavy.** Shopify, Givex (gift cards and loyalty), Amazon Marketplace, 3PL partners and wholesale retail partners all hold or exchange company data, so supply-chain and SaaS security matter as much as on-premises controls.

- **Quebec Law 25 is the strictest privacy law that applies.** As a Quebec company it must meet Law 25 (fully in force since September 2024) as well as PIPEDA, PCI DSS for card data, CASL for marketing e-mail and U.S. state privacy laws for U.S. customers.

- **Design direction.** The Lab 2 zoned design (Users, Admin, DMZ, Internal, Infrastructure) maps directly onto DAVIDsTEA's business and will be extended in Stage 2 with store/POS, warehouse and management zones.

# 2. Company Selection

## 2.1 Selection criteria

The project requires a fictitious evaluation of a real Canadian mid-sized organisation. We used the following criteria to choose a company:

| **Criterion** | **Why it matters** | **DAVIDsTEA** |
|----|----|----|
| **Canadian, mid-sized** | Matches the project scope (roughly 100–500 staff) | Montréal HQ; ~250 full-time staff; \$61.0M sales |
| **Rich public information** | OSINT must be possible without touching company assets | Former Nasdaq/SEC filer, now TSX Venture; detailed 10-K and press releases |
| **Public-facing services** | Needed for DMZ, WAF and external testing design | E-commerce store, investor site, Amazon storefront, loyalty program |
| **Internal IT services** | Needed for AD, DNS, DHCP, ERP, file-share design | Head office, production, warehouse, 21 stores |
| **Regulated data** | Gives meaningful compliance requirements | Payment cards, loyalty PII, Quebec residents' data |
| **Fits our lab platform** | Lets Stages 3–4 reuse Lab 1/Lab 2 work | Web store, ERP, AD and file shares map to PrestaShop, Odoo, AD and Samba |

## 2.2 Alternatives considered

- **Kicking Horse Coffee —** rejected: owned by Lavazza since 2017, so most IT is run by a foreign parent.

- **A regional credit union —** rejected: very little technical information is public and banking regulation (OSFI) would dominate the project.

- **A municipal government —** rejected: public sector, not a business, and different privacy laws apply.

# 3. Company Profile

| **Attribute** | **Detail** | **Source** |
|----|----|----|
| **Legal name** | DAVIDsTEA Inc. (Canada Business Corporations Act) | \[1\] |
| **Industry** | Specialty retail and wholesale — loose-leaf tea, tea accessories and gifts | \[1\]\[2\] |
| **Headquarters** | 5430 Ferrier, Mount-Royal (Montréal), Québec H4P 1M2 (address as last filed) | \[1\] |
| **Founded** | 2008, Toronto (first store); headquartered in Montréal | \[1\] |
| **Listing** | TSX Venture Exchange (DTEA); voluntarily delisted from Nasdaq in April 2023 and deregistered with the SEC | \[3\] |
| **Employees** | About 250 full-time plus part-time store staff (252 FT / 182 PT at last detailed disclosure) | \[1\]\[4\] |
| **Fiscal 2025 sales** | \$61.0M (year ended January 31, 2026); net income \$2.9M; adjusted EBITDA \$7.6M | \[2\] |
| **Sales channels** | Online 46.9% (\$28.6M) · Retail 39.8% (\$24.3M) · Wholesale 13.2% (\$8.1M) | \[2\] |
| **Geography** | Canada \$53.8M, United States \$7.2M | \[2\] |
| **Stores** | 21 company-owned stores in Canada; target of 25 by end of fiscal 2026 (Oshawa, Mississauga, Edmonton, Vancouver) | \[2\] |
| **Wholesale reach** | 4,000+ grocery stores and pharmacies and 1,500+ convenience stores in Canada; 900+ grocery stores in the U.S. | \[2\] |
| **Operations** | Production facility in Montréal; Canadian fulfilment (Sherbrooke, QC historically, now internalised); U.S. orders shipped by a 3PL in Chicago since March 2026 | \[1\]\[2\] |
| **Loyalty** | Frequent Steeper / Super Steeper loyalty program | \[1\] |
| **History** | Restructured under the CCAA in 2020 (closed all U.S. and most Canadian stores and pivoted to online); re-platformed IT in fiscal 2024–2025 | \[1\]\[2\] |

## 3.1 Business functions that depend on IT

- Online ordering, payment, order tracking and customer accounts (web store and Amazon).

- In-store sales, returns, gift cards and loyalty (POS in 21 stores).

- Wholesale order management and EDI-style exchanges with grocery and pharmacy chains.

- Production planning, inventory, warehouse management and shipping to customers and 3PLs.

- Finance, accounting, payroll and HR for roughly 400 full- and part-time staff.

- Marketing: e-mail campaigns, loyalty communications, social media and customer service.

- Corporate: investor relations, continuous disclosure to the TSXV, legal and board communications.

# 4. OSINT Methodology and Rules of Engagement

## 4.1 Rules of engagement

- **No traffic to company assets.** We did not browse, scan, ping, resolve or crawl any DAVIDsTEA domain or IP address. All information came from third-party sources (regulators, news, job boards, vendor case studies).

- **No active techniques.** No port scanning, banner grabbing, directory brute-forcing, phishing or social engineering of employees.

- **Legal compliance.** Activities stayed within the Criminal Code (s. 342.1 unauthorised use of a computer, s. 430(1.1) mischief in relation to computer data) and PIPEDA. Personal information about individual employees was not collected.

- **Verification labels.** Every technical statement in this report is labelled Confirmed (public source names it) or Inferred (reasoned from the business model or common practice). Inferred items must be validated before being treated as fact.

## 4.2 Sources used

| **Source type** | **Examples** | **What it provided** |
|----|----|----|
| **Regulatory filings** | SEC EDGAR 10-K / 10-Q / 8-K (to 2023), SEDAR+ | Employees, facilities, IT and cyber risk factors, insurance |
| **Press releases** | Newsfile fiscal 2025 results (April 2026) | Sales by channel, store count, IT re-platforming, 3PL changes |
| **Job postings** | Espresso-jobs (UX Engineer, Shopify/e-commerce) | Shopify Plus, Shopify CLI, Git workflows |
| **Vendor / agency case studies** | Diff Agency DAVIDsTEA project page | Migration from Salesforce Commerce Cloud to Shopify Plus, Shopify POS, Givex |
| **Trade press** | Retail Insider | Strategy, store openings, financial context |
| **Legislation** | openparliament.ca, Gowling WLG, Quebec CAI guidance | Bill C-8, Bill C-27 status, Law 25 obligations |

## 4.3 Further passive sources for later stages

The following passive sources can enrich Stage 2 without contacting company assets: certificate-transparency logs (crt.sh) for subdomains, historical passive-DNS datasets, the Wayback Machine for old site technology, Have I Been Pwned domain search (only with company authorisation), LinkedIn job titles for tool names, and Shopify's public app store reviews.

# 5. Public-Facing Services

These services are reachable from the Internet by customers, partners or investors. They form the external attack surface and will be placed in, or protected from, the DMZ and cloud zones in the security architecture.

| **Service** | **Platform / provider** | **Purpose** | **Data handled** | **Status** |
|----|----|----|----|----|
| **E-commerce store (davidstea.com, EN/FR)** | Shopify Plus (SaaS), custom theme | Online catalogue, checkout, customer accounts, pre-orders, bundles | Customer PII, addresses, order history; card data tokenised by Shopify Payments / gateway | Confirmed \[5\]\[6\] |
| **Customer accounts and login** | Shopify customer accounts | Order history, saved addresses, loyalty balance | Credentials or one-time codes, PII | Confirmed (platform) / Inferred (details) |
| **Loyalty program (Frequent Steeper)** | Givex integration + Shopify | Points, rewards, member offers | Member PII, purchase history | Confirmed \[1\]\[6\] |
| **Gift cards (online and in-store)** | Givex | Sell and redeem gift cards | Card numbers / balances (fraud target) | Confirmed \[6\] |
| **Amazon Marketplace storefront** | Amazon Seller Central | Selected assortment sold through Amazon | Order data shared with Amazon | Confirmed \[1\]\[2\] |
| **Investor relations site** | ir.davidstea.com (hosting provider not verified) | Filings, press releases, AGM material | Public data; e-mail alert subscribers | Confirmed (exists) \[3\] |
| **Corporate e-mail (MX)** | Not verified — likely Microsoft 365 or Google Workspace | Staff, customer service, supplier e-mail | Business data, invoices; phishing entry point | Inferred |
| **Marketing e-mail / SMS** | E-mail service provider (ESP) — not verified | Newsletters, loyalty offers | Subscriber lists, consent records (CASL) | Inferred |
| **Customer service** | Help desk / chat SaaS — not verified | Order issues, returns | PII, order details | Inferred |
| **Careers / recruiting** | Third-party job boards and ATS | Hiring for stores and head office | Applicant résumés (PII) | Confirmed (postings) / Inferred (ATS) |
| **Wholesale partner exchange** | EDI / supplier portals of grocery chains | Purchase orders, invoices, ASN | Commercial data | Inferred |
| **Social media** | Instagram, Facebook, TikTok, etc. | Marketing, brand | Account takeover risk | Inferred |

*Table 5-1 — Public-facing services. Bracketed numbers refer to Section 11.*

# 6. Internal IT Infrastructure Services

DAVIDsTEA has not published a network diagram, so the internal services below are inferred from its 10-K disclosures ("industry-standard and customized software systems" for point of sale, inventory, warehouse management and accounting \[1\]), from the 2025 re-platforming, and from what a company of this size and type needs. Each item becomes a requirement for the Stage 2 infrastructure catalogue.

| **Service** | **Likely implementation** | **Users** | **Basis** | **Lab equivalent** |
|----|----|----|----|----|
| **Identity & directory** | Microsoft Entra ID / Active Directory, SSO to SaaS apps | All staff | Inferred | AD DS on Windows Server 2019 |
| **DNS (internal and public)** | Internal AD DNS; public DNS hosted by registrar/CDN | All | Inferred | PrivateDNS + PublicDNS |
| **DHCP** | Per-site DHCP (HQ, production, stores) | Workstations, POS, printers, Wi-Fi | Inferred | DHCP server + Core relay |
| **ERP / finance / inventory** | Cloud ERP after re-platforming (vendor not public) | Finance, buying, operations | Confirmed (re-platform) \[2\] / vendor inferred | Odoo |
| **Point of sale** | Shopify POS on tablets/terminals; card terminals from payment processor | Store staff | Confirmed \[6\] | Simulated POS VLAN |
| **Warehouse management** | WMS linked to ERP and Shopify; integration with Chicago 3PL | Warehouse, logistics | Confirmed (WMS use) \[1\] | Odoo Inventory module |
| **File sharing** | SharePoint/OneDrive or on-prem file server | Head office | Inferred | Samba |
| **E-mail & collaboration** | Microsoft 365 or Google Workspace, Teams/Meet | All staff | Inferred | — |
| **HR & payroll** | SaaS payroll (common Canadian providers) | HR, all staff | Inferred | — |
| **Integration / middleware** | Connectors between Shopify, Givex, ERP, Amazon and 3PL | IT | Inferred | — |
| **Production systems** | Recipe, batch and labelling systems at Montréal facility | Production staff | Inferred | — |
| **Network** | SD-WAN or VPN from 21 stores to HQ/cloud; guest Wi-Fi in stores | All sites | Inferred | Edge/Core routers, OSPF |
| **Endpoint & security tools** | EDR/antivirus, MDM for store tablets, backup, logging | IT | Inferred | nftables, Wazuh/Suricata (planned) |
| **Remote access** | VPN/ZTNA for head-office and IT admins | Office staff, IT | Inferred | Admin VLAN / jump host |

*Table 6-1 — Internal IT services (confirmed and inferred).*

# 7. Technical Specifications

## 7.1 Known and probable technology stack

| **Layer** | **Technology** | **Confidence** | **Security notes** |
|----|----|----|----|
| **E-commerce** | Shopify Plus (hosted SaaS), Liquid theme, metafields/metaobjects, Shopify CLI + Git | Confirmed \[5\]\[6\] | Shopify is PCI DSS Level 1; risk shifts to apps, scripts, admin accounts and API keys |
| **Previous e-commerce** | Salesforce Commerce Cloud (retired) | Confirmed \[6\] | Old data, accounts and integrations must be decommissioned |
| **Point of sale** | Shopify POS + POS UI extensions | Confirmed \[6\] | Tablet hardening, MDM, P2PE card readers |
| **Gift card / loyalty** | Givex | Confirmed \[6\] | API keys, gift-card fraud, loyalty account takeover |
| **Marketplace** | Amazon Seller Central | Confirmed \[1\] | MFA on seller accounts, shared order data |
| **Payments** | Shopify Payments or third-party gateway; in-store terminals | Inferred | Card data out of scope if fully tokenised; SAQ A / SAQ P2PE |
| **ERP / WMS** | Cloud ERP after fiscal 2024 re-platforming (\$3.1M legacy contract exit) | Confirmed (change) / Inferred (vendor) | SSO, least privilege, segregation of duties |
| **Productivity** | Microsoft 365 or Google Workspace | Inferred | MFA, conditional access, e-mail security (SPF/DKIM/DMARC) |
| **Endpoints** | Windows 10/11 laptops at HQ; iPads/terminals in stores | Inferred | EDR, patching, MDM |
| **Network** | Store broadband + LTE backup, VPN/SD-WAN, firewalls at HQ | Inferred | Segment POS, guest Wi-Fi and corporate traffic |
| **Hosting** | Mostly SaaS/cloud after re-platforming; small on-prem footprint at HQ | Inferred | Shared-responsibility model; logging from SaaS |

*Table 7-1 — Technology stack.*

## 7.2 Estimated asset inventory

Estimates based on staff numbers and store count; to be refined in Stage 3.

| **Site** | **Count** | **Typical assets** |
|----|----|----|
| **Head office, Mount-Royal QC** | 1 | ~120–140 staff laptops, printers, meeting-room devices, core switches, firewall, Wi-Fi, small server room |
| **Production facility, Montréal** | 1 | Shop-floor PCs, label printers, scanners, network switch |
| **Retail stores** | 21 (25 planned) | 2–4 POS devices, card readers, back-office PC, router/firewall, staff and guest Wi-Fi each |
| **Fulfilment / 3PL** | Canada internal + Chicago 3PL | Scanners, label printers, WMS terminals; 3PL systems integrated via API |
| **Cloud / SaaS** | 8–15 services | Shopify, Givex, Amazon, ERP, M365/Google, ESP, help desk, HR/payroll, backup |

# 8. Initial Threat Landscape

These threats are typical of omni-channel retailers and will feed the NIST SP 800-30 Rev. 1 risk assessment (TVRA) in Stage 4.

| **Threat** | **Target asset** | **Example / relevance** | **Initial rating** |
|----|----|----|----|
| **Web skimming (Magecart) via third-party scripts or apps** | Web store checkout | Malicious JavaScript in themes or apps steals card data | High |
| **Credential stuffing / account takeover** | Customer and loyalty accounts | Reused passwords; stolen points and gift-card balances | High |
| **Phishing and business e-mail compromise** | Staff e-mail, finance | Fake supplier invoices, payroll diversion | High |
| **Ransomware** | HQ endpoints, file shares, ERP | Retail is a frequent ransomware target; stops fulfilment | High |
| **Third-party / SaaS compromise** | Shopify apps, Givex, 3PL, ESP | Supplier breach exposes customer data | Medium–High |
| **Gift-card fraud and enumeration** | Givex gift cards | Brute-forcing card numbers, refund abuse | Medium |
| **POS and store network attacks** | Store Wi-Fi, POS tablets | Weak guest/POS separation, stolen devices | Medium |
| **Insider misuse** | ERP, customer data | Excessive privileges, departing staff | Medium |
| **DDoS / bot traffic** | Web store during promotions | Mitigated mostly by Shopify; bots buy limited stock | Low–Medium |

*Table 8-1 — Initial threats (ratings are qualitative and will be revised in the TVRA).*

# 9. Laws, Regulations and Pending Legislation

## 9.1 In force

| **Law / standard** | **Applies because** | **Key requirements for the security design** |
|----|----|----|
| **Quebec Law 25 (Act respecting the protection of personal information in the private sector, as amended)** | Headquartered in Quebec; holds data of Quebec residents. Fully in force since September 22, 2024. | Named privacy officer; privacy impact assessments for new systems and transfers outside Quebec; register of confidentiality incidents; notify the CAI and individuals of incidents with risk of serious injury; privacy by default; data portability; penalties up to \$25M or 4% of worldwide turnover. |
| **PIPEDA (federal)** | Commercial activity across provinces; customers in other provinces. | Safeguards appropriate to sensitivity (Principle 4.7); breach of security safeguards reporting to the OPC and affected people; keep breach records for 24 months. |
| **PCI DSS v4.0.1** | Accepts Visa/Mastercard online and in store (contractual via acquirer). | Reduce scope with tokenisation and P2PE; MFA for admin access; script inventory and integrity for payment pages (req. 6.4.3 and 11.6.1); quarterly scans; annual SAQ. |
| **CASL — Canada's Anti-Spam Legislation** | Sends marketing e-mail and SMS to loyalty members. | Consent records, unsubscribe in 10 business days; also bans installing software or altering data without consent. |
| **Criminal Code of Canada (ss. 342.1, 430(1.1))** | Applies to attackers and to our own testing. | Testing only on our own lab systems; logging and evidence handling to support prosecution. |
| **Securities law — TSXV and CSA Staff Notice 51-347 / NI 51-102** | Listed issuer on the TSX Venture Exchange. | Timely disclosure of material cyber incidents; board oversight of cyber risk. |
| **U.S. state privacy laws (e.g., California CCPA/CPRA) and breach-notification laws** | \$7.2M of sales to U.S. customers. | Reasonable security, notice to residents after a breach, consumer rights requests. |
| **Charter of the French Language (Bill 96)** | Quebec business. | Systems, interfaces and documentation available in French — affects user-facing security notices and policies. |

*Table 9-1 — Laws and standards in force.*

## 9.2 Pending or proposed legislation

| **Bill** | **Status (October 2026)** | **Impact on DAVIDsTEA** |
|----|----|----|
| **Bill C-8 — Critical Cyber Systems Protection Act (CCSPA) and Telecommunications Act amendments** | Reported as receiving Royal Assent in June 2026; Part 2 (CCSPA) not yet in force and no operators designated. | A tea retailer is not a designated critical-infrastructure operator, but its telecom, payment and banking suppliers may be. Expect stricter security clauses in supplier contracts. |
| **Federal privacy reform (successor to Bill C-27 / Consumer Privacy Protection Act)** | Bill C-27 died when Parliament was prorogued in January 2025; government has signalled reintroduction. | Would replace PIPEDA with fines up to 5% of global revenue and a privacy tribunal. Designing for Law 25 now covers most expected obligations. |
| **Artificial Intelligence and Data Act (AIDA)** | Died with Bill C-27. | Relevant only if AI is used for personalised offers; monitor. |

*Table 9-2 — Pending legislation. Status to be re-checked before final submission.*

## 9.3 Frameworks we will use

- **CIS Critical Security Controls v8.1 —** to assign controls to each security zone (as required by the project).

- **NIST SP 800-30 Rev. 1 —** for the TVRA risk assessment.

- **NIST CSF 2.0 —** to organise controls into Govern, Identify, Protect, Detect, Respond and Recover.

- **Canadian Centre for Cyber Security — Baseline Cyber Security Controls for Small and Medium Organizations —** as a Canadian baseline.

# 10. Proposed Scope for the Security Architecture

## 10.1 Mapping DAVIDsTEA to our lab environment

Our lab will model DAVIDsTEA's head office and one representative store. SaaS services that we cannot run (Shopify, Givex, Amazon) are simulated with open-source equivalents so they can be secured and tested.

| **Zone (Lab 2 basis)** | **DAVIDsTEA function** | **Lab systems** |
|----|----|----|
| **Outside / Internet** | Customers, attackers, suppliers | Outside Windows client, Kali |
| **DMZ** | Web store, corporate/IR site, public DNS | PrestaShop (web store), Joomla (IR site), PublicDNS |
| **Internal applications** | ERP, inventory, warehouse, file shares | Odoo, Samba |
| **Infrastructure** | Identity, DNS, DHCP, logging | AD DS, DHCP, PrivateDNS (+ SIEM planned) |
| **Users** | Head-office staff | Windows 10 domain clients |
| **Admin / management** | IT administrators | Admin workstation, jump host |
| **Store / POS (new)** | Retail store POS and back office | POS VLAN with simulated terminals |
| **Guest Wi-Fi (new)** | Customers in stores | Isolated VLAN, Internet only |

## 10.2 Preliminary security requirements

- Default-deny firewall policy between all zones; only documented flows allowed.

- Web application firewall and IPS in front of the web store and IR site.

- Centralised identity with MFA and single sign-on; role-based access control for store, office, finance and IT roles.

- Card data never stored; POS zone isolated from corporate and guest networks.

- Encrypted traffic (HTTPS/TLS 1.2+) for all public and admin services.

- Central logging and alerting (SIEM) with retention that meets Law 25 incident-register needs.

- Backups of ERP, AD and file shares, tested for ransomware recovery.

- Vulnerability scanning of all assets and a documented patch process.

- Incident response plan with Law 25, PIPEDA and PCI notification steps.

## 10.3 Assumptions and limitations

- Inferred items are reasonable assumptions, not facts; DAVIDsTEA has not confirmed them.

- The most detailed IT disclosures date from the 2021 10-K; some facts (addresses, distribution sites) may have changed.

- Store and asset counts are estimates for design purposes.

- This is an academic exercise; the evaluation is fictitious and no company systems were touched.

# 11. References

\[1\] DAVIDsTEA Inc. (2021). Form 10-K for the fiscal year ended January 30, 2021. U.S. SEC EDGAR. https://www.sec.gov/Archives/edgar/data/1627606/000147793221002796/dtea_10k.htm

\[2\] DAVIDsTEA Inc. (2026, April 29). DAVIDsTEA returns to profitability in fiscal 2025, reports strong fourth quarter results. Newsfile. https://www.newsfilecorp.com/release/294800

\[3\] DAVIDsTEA Inc. (2023). DAVIDsTEA to cease trading on Nasdaq at the close of market on Friday, April 14, 2023 (press release, as reported via SEC 8-K filings). https://www.sec.gov/cgi-bin/browse-edgar?action=getcompany&CIK=1627606

\[4\] roic.ai. (n.d.). DAVIDsTEA Inc. (DTEAF) company profile. https://www.roic.ai/quote/DTEAF

\[5\] Espresso-jobs. (2026). UX Engineer (Shopify / Ecommerce) — DAVIDsTEA Inc., Mount-Royal. https://www.espresso-jobs.com/emploi/214672/ux-engineer-shopify-ecommerce-davidstea-inc-mount-royal

\[6\] Diff Agency. (n.d.). DAVIDsTEA project showcase. https://www.diffagency.com/works/davidstea

\[7\] Retail Insider. (2025, June). DAVIDsTEA delivers strong start to fiscal 2025. https://retail-insider.com/retail-insider/2025/06/davidstea-delivers-strong-start-to-fiscal-2025-with-higher-margins-and-profitability-gains-in-q1/

\[8\] OpenParliament.ca. (2026). Bill C-8 (45-1): An Act respecting cyber security. https://openparliament.ca/bills/45-1/C-8/

\[9\] Gowling WLG. (2025). Federal privacy reform / Bill C-27 timeline of developments. https://gowlingwlg.com/en/insights-resources/articles/2025/federal-privacy-reform

\[10\] Commission d'accès à l'information du Québec. Law 25 guidance. https://www.cai.gouv.qc.ca/

\[11\] Office of the Privacy Commissioner of Canada. PIPEDA requirements in brief. https://www.priv.gc.ca/

\[12\] PCI Security Standards Council. (2024). PCI DSS v4.0.1. https://www.pcisecuritystandards.org/

\[13\] Center for Internet Security. (2024). CIS Critical Security Controls v8.1. https://www.cisecurity.org/controls

\[14\] NIST. (2012). SP 800-30 Rev. 1, Guide for Conducting Risk Assessments. https://csrc.nist.gov/pubs/sp/800/30/r1/final

\[15\] Canadian Centre for Cyber Security. Baseline cyber security controls for small and medium organizations. https://www.cyber.gc.ca/
