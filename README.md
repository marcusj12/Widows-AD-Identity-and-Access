# Active Directory Home Lab — Learning How Identity and Access Actually Work

> A self-directed lab I'm building to understand — not just configure — how directory services, authentication, and access control fit together in a Windows environment. I'm preparing for a role in IT/systems administration, and I wanted the concepts I'm studying to become something I could reason about, not just memorize.
---

## Why I built this

I kept running into the same wall while studying: I could recite what Active Directory, DNS, DHCP, and Group Policy *where*, but I couldn't really explain how they depended on each other, or what actually happens when a user logs in. Definitions weren't turning into understanding.

So I built the thing from scratch. One Windows Server, a fake company (`corp.marc.com`), and a rule for myself: **don't move to the next step until I can explain why the last one was necessary.** This repo is the record of that — what I set up, what broke, what confused me, and what finally made it click.

It's a work in progress on purpose. Part 1 (below) builds the foundation of *identity* — who exists on the network. Part 2 is where the study gets to the part I actually care about most: *access* — passwords, permissions, and how credentials are controlled.

---

## The lab

| Piece | What it is | Value |
|---|---|---|
| **NB-DCO1** | The domain controller — the "gatekeeper" holding all accounts | `192.168.163.133` (static) |
| **`corp.marc.com`** | The domain — the whole network "kingdom" | NetBIOS: `CORP` |
| **Network** | Isolated lab subnet in VirtualBox | `192.168.163.0/24` |

<img width="2720" height="2000" alt="ad_lab_network_topology" src="https://github.com/user-attachments/assets/b9e53c4a-2ad9-4173-8fd2-6f754650710f" />

---
## The Setup: Getting Things Ready:
### Setting Up Server 2022
<img width="965" height="712" alt="Screenshot 2026-07-14 at 11 33 11 PM" src="https://github.com/user-attachments/assets/452a4737-77c2-4e16-989f-d1ebb8229908" />

### Setting Up Active Directory
*Notice that the Orginal Server name was WIN-LAAM34FL0AK, later changed to NB-DC01*
<img width="1011" height="771" alt="Screenshot 2026-07-15 at 7 33 05 PM" src="https://github.com/user-attachments/assets/c9b6717c-1c19-452a-80d0-5a366e0ea506" />

<img width="1390" height="775" alt="AD DS Conf" src="https://github.com/user-attachments/assets/1391114d-0c15-4499-929d-59188066ae84" />





## Part 1 — Building identity (what I did, and what each step taught me)

### Standing up the domain controller
I installed Windows Server 2022 and promoted it to a domain controller. The first thing that actually taught me something wasn't a feature — it was being forced to give the server a **static IP**.

**What clicked:** every other machine on the network is going to be told "ask this server to log you in and to look up names." If that server's address could change out from under them, the whole network would break. Infrastructure has to sit still. That's the difference between a server and a client, and I didn't really *get* it until I had to configure it.

<img width="1284" height="871" alt="Server Static IP Conf" src="https://github.com/user-attachments/assets/3dd2cf15-2159-4188-9cf3-b922a9607116" />

<img width="1123" height="670" alt="Screenshot 2026-07-26 at 7 46 10 PM" src="https://github.com/user-attachments/assets/19ca9eca-3af1-4e06-a3c4-44d46a0d9b69" />


### DNS — the piece I didn't know I already had
At one point I panicked because I didn't remember ever installing DNS. It turned out DNS had been installed automatically when I promoted the domain controller.

**What clicked:** a Windows domain *cannot exist* without DNS — it's how clients even find the domain controller to authenticate against. DNS isn't a separate add-on to Active Directory; it's the foundation AD stands on. Realizing that reframed how I think about the whole stack: names have to resolve before anything else can happen.

<img width="1494" height="932" alt="ADDomain conf 1 4" src="https://github.com/user-attachments/assets/360ea563-fcf7-473b-9dfc-cf157d4879b4" />


### DHCP — and why the network wouldn't just let me turn it on
I had DHCP installed but it refused to hand out addresses. The error said it wasn't "authorized."

**Why: In a domain, Windows refuses to let a DHCP server hand out addresses until it's been “authorized” in Active Directory. This is a safety feature that blocks rogue DHCP servers from disrupting a network.**


**What clicked:** in a domain, Windows deliberately blocks a DHCP server from working until it's been authorized in Active Directory. It's a security control — it stops a rogue DHCP server from hijacking the network by handing out bad configuration. That was the first moment access control showed up as a *theme*: the system defaults to "no" and makes you prove you're allowed. I authorized it, created the address pool, and set the options that tell every client where to find DNS and the gateway.

#### Create Address Pool

<img width="1885" height="1052" alt="Create Address Pool3 2" src="https://github.com/user-attachments/assets/63c3a55f-57f7-4e9b-80bc-439b2691188a" />

##### DHCP Scope Completion:

<img width="1912" height="1052" alt="DHCP Scope Completion3 2 1" src="https://github.com/user-attachments/assets/00bf1559-f4a9-4bf1-93a2-17895c062bdf" />

##### Powershell (CLI) DHCP server scope confirmation:![Uploading DHCP Scope Completion3.2.1.png…]()

<img width="1502" height="312" alt="PowerShell CLI Scop Conf3 2 2" src="https://github.com/user-attachments/assets/e1e02ebf-2393-4a29-9032-25ae00948f0b" />

### Users, groups, and structure — where automation started to make sense
I built an organized folder structure (OUs) for departments, then created users. Doing the first few by hand in the GUI was tedious; doing 20 that way would have been miserable.

**What clicked:** this is *why* PowerShell exists in this world. I wrote a script that reads a spreadsheet and creates every account with a consistent username scheme, drops each person into their department's folder, and forces a password reset on first login. Then I built security groups and had a script auto-fill them by reading each user's department. Twenty users, correct and identical, in seconds. Automation stopped being a buzzword and became an obvious answer to a real problem I'd just felt.

#### Create a users.csv file
<img width="1095" height="666" alt="UserNameCsv4 2" src="https://github.com/user-attachments/assets/5b514a92-86f1-4174-a858-52e3ae5daeda" />

#### Successful Organizational Unit (OU) and Corp Org Tree Setup:
*Why: A tidy OU tree is how real admins keep a network manageable. Dumping everyone in the default “Users” container is a rookie tell; a departmental structure lets you target policies and delegate control cleanly*

<img width="1484" height="896" alt="OU Tree Setup4 1" src="https://github.com/user-attachments/assets/26e0c0d7-64c0-4050-9e05-e2dc3d9a074a" />

<img width="1180" height="707" alt="CorpOrg Tree4 1 2" src="https://github.com/user-attachments/assets/522033bb-eb30-463c-84dc-4bb0bfa5b381" />

#### Python Script to import users from user.csv, build a Username, create a account and account password parameters:
*Why: Creating 15+  users by hand takes an hour of clicking; this script does it in seconds and never makes a naming mistake. “Automated user provisioning from CSV with PowerShell”*

<img width="1339" height="990" alt="Screenshot 2026-08-11 at 8 53 15 PM" src="https://github.com/user-attachments/assets/28125c79-b49b-4fe0-ae98-a7c45d30ca18" />

#### CLI AD User Check:

<img width="1278" height="722" alt="ADUserCheck4 2 2" src="https://github.com/user-attachments/assets/275084a3-e834-4042-b70f-5bd280e68c06" />

#### Python Automation Script for adding user to Security Groups:

<img width="1096" height="790" alt="CreateSecGroupScript4 3" src="https://github.com/user-attachments/assets/87309693-6166-4ecd-9ec7-291b56e1c638" />

I redundantly entered the same came cmmd that ran the powershell script containing instuctions to assign users groups to the appropriate Department

<img width="1116" height="597" alt="Redundant Script 4 3 2" src="https://github.com/user-attachments/assets/0482f780-23ea-4b1d-a5b5-cb1494997ff6" />

#### UserName Check:

<img width="1188" height="645" alt="UserNameDepCheck4 3 3" src="https://github.com/user-attachments/assets/91f29598-4e17-4020-b3af-fad0c2b14a43" />

#### Full Directory Check:

<img width="1202" height="702" alt="FullDirectoryCheck4 3 4" src="https://github.com/user-attachments/assets/b7ece018-7855-44bb-bf03-cc0e8a70e4f3" />


---

## Part 2 — Building access (what I'm studying next)

This is the half I started the project for. Part 1 answered *who exists*. Part 2 is about *what they're allowed to do* — and it's where the security concepts I'm studying live.

I want to come out of this able to explain, not just click:

- **Passwords & authentication policy** — how password rules, lockout, and complexity are enforced across a whole domain from one place, and *why* centralizing that matters for security.
- **Group Policy** — how a single policy can shape thousands of machines: screen locks, hardening, restrictions. I want to understand the enforcement model (what applies where, and in what order), not just toggle settings.
- **NTFS & share permissions** — the actual mechanics of access control: how "Finance can open Finance files and no one else can" is really enforced, and the classic distinction between share-level and NTFS permissions.
- **How credentials flow** — what genuinely happens between typing a password and getting access. Kerberos, tickets, and why groups (not individual users) are the unit of access control.

The through-line I'm chasing: **identity, authentication, and authorization are three different things**, and this lab is where I'm learning to feel the difference instead of reciting it.

Planned build: join a client to the domain → enforce password/lockout policy via Group Policy → harden the workstation → stand up a file server with department-restricted permissions → trace how a login actually resolves to access.

---

## Troubleshooting log — the part that taught me the most

Honestly, I learned more from things breaking than from things working. A sample:

| What went wrong | What I thought | What was actually happening |
|---|---|---|
| DHCP wouldn't service clients (Event 1046/1059) | "It's broken" | It was doing its job — refusing to run until authorized in AD (a security guardrail) |
| `Add-DhcpServerInDC` threw RPC error 1722 | "It failed" | It had *succeeded*; the service just needed a restart to notice |
| `Resolve-DnsName NB-DC01...` said "does not exist" | "DNS is broken" | The hostname is `NB-DCO1` — a letter **O**, not a zero. One character cost me an hour |
| A stale DNS record pointed at an old machine name | confusion | The server was renamed after first boot; the old record lingered and I learned to clean it up |
| `New-ADUser` rejected my script | "the script is wrong" | Typo — `-SameAccountName` instead of `-SamAccountName`. Precision is the skill |

The meta-lesson: **most "errors" are messages, not disasters.** Reading them slowly, instead of panicking, is most of the job.

---

## What this project is really teaching me

- The pieces aren't separate. DNS lets clients find the DC; the DC authenticates them; DHCP hands them the address and points them at DNS. Break one and the rest fall over. I can finally draw that dependency chain from memory.
- Security is a default posture, not a feature you add. The system kept saying "no" until I proved authorization — DHCP, secure DNS updates, password-at-logon. That mindset is the point.
- Automation is a response to real pain, not a flex.
- Careful reading beats fast clicking. Every hard bug this project threw at me was a detail I skimmed.

---

## Repo structure

```
.
├── README.md
├── scripts/
│   ├── Cre.ps1
│   ├── CreateSecGroups.ps1
│   └── users.csv
└── screenshots/
    ├── architecture.png
    ├── phase1-dcdiag.png
    ├── phase2-dns-zone.png
    ├── phase3-dhcp-scope.png
    └── phase4-users-groups.png
```

## Notes

Isolated lab environment for study. The domain, users, and credentials are fictional and live only inside a local VM. Any passwords in scripts are throwaway lab values, changed on first logon.
