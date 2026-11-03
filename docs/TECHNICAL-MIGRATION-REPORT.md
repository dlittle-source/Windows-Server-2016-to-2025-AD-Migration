# Technical Migration Report

## Enterprise Active Directory Modernization — Windows Server 2016 to Windows Server 2025

**Environment:** Isolated Oracle VirtualBox lab  
**Domain:** `migration.test`  
**Migration approach:** Side-by-side addition of a Windows Server 2025 domain controller, FSMO transfer, validation, and graceful demotion of the Windows Server 2016 domain controller  
**Execution date:** October 31, 2026  
**Evidence:** [Migration screenshots](../screenshots/)  
**Companion document:** [Project README](../README.md)

> **Scope and evidence statement:** This report describes a hands-on laboratory migration, not a production change. Commands and outcomes below are based on the recorded migration session and captured screenshots. Where a check was not documented or a warning remained, the report says so explicitly. Command blocks are illustrative of commands executed or recorded; they are not a single unattended migration script.

---

## 1. Executive technical summary

The `migration.test` Active Directory domain originally ran on **MIG-DC01**, a Windows Server 2016 domain controller. A Windows Server 2025 member server, **MIG-DC02**, was promoted as an additional writable domain controller with DNS and Global Catalog capabilities. Directory replication and SYSVOL availability were checked; all five FSMO roles were transferred to MIG-DC02; and client authentication was tested while the original controller was unavailable.

During the retirement phase, the initial normal demotion of MIG-DC01 failed because directory partition replication could not complete. Investigation found **replication error 8524 (DNS lookup failure)** for `ForestDnsZones`. After changing MIG-DC01's DNS client configuration to use MIG-DC02 and successfully replicating that partition, the normal demotion completed. Final directory enumeration and DC locator DNS queries showed MIG-DC02 as the sole domain controller.

**Verified outcome:** Domain controller services moved to Windows Server 2025; all five FSMO roles were held by MIG-DC02; an uncached first sign-in succeeded through MIG-DC02; and MIG-DC01 was gracefully demoted. **Caveat:** Post-migration `dcdiag /q` still reported DFSR/SystemLog events, so this report does not assert that all diagnostic checks were clean.

## 2. Environment and architecture

| Component | Before migration | After migration |
|---|---|---|
| AD domain | `migration.test` | `migration.test` |
| Domain controller | `MIG-DC01` — Windows Server 2016 | `MIG-DC02` — Windows Server 2025 |
| IP address | `192.168.200.10` | `192.168.200.11` |
| FSMO roles | All five on MIG-DC01 | All five on MIG-DC02 |
| DNS / Global Catalog | MIG-DC01 | MIG-DC02 |
| Domain-joined clients | `MIG-CL01`, `MIG-CL02` — Windows 11 | Same clients; MIG-CL01 explicitly validated against MIG-DC02 |
| Virtual network | VirtualBox `NatNetwork`, `192.168.200.0/24` | Same |
| Virtual gateway | `192.168.200.1` | Same |

The NAT Network DHCP range was configured for `192.168.200.100–192.168.200.200`. MIG-CL01 initially received `.100` and MIG-CL02 `.101`. Both controllers used static addresses.

**Functional-level note:** The domain and forest were created at the Windows Server 2016 functional level. An increase to a Windows Server 2025 functional level was **not verified** and is **not claimed**. The AD schema version was observed as **91** after preparation, compared with version **87** before migration.

## 3. Pre-migration assessment and recovery preparation

### 3.1 Service and directory health

On MIG-DC01, the core services were verified:

```powershell
Get-Service -Name NTDS,DNS,KDC,Netlogon
```

**Observed:** All four services were running. The following checks were also used:

```powershell
dcdiag /q
net share
netdom query fsmo
Get-CimInstance -Namespace root\microsoftdfs -ClassName DfsrReplicatedFolderInfo
```

**Observed:** `SYSVOL` and `NETLOGON` shares existed; the DFSR-replicated SYSVOL folder reported **State 4 (Normal)**; and all five FSMO roles initially belonged to MIG-DC01. The baseline `dcdiag /q` produced **DFSREvent and SystemLog warnings/errors**, associated with earlier unexpected VM shutdown and service recovery. The report therefore does **not** label the baseline diagnostic output as clean.

MIG-DC01 memory was increased from **2 GB to 4 GB** following lab stability issues. DFS Replication recovery events included **2212, 2218, and 2214**; recovery was observed before proceeding.

**Evidence:** [01 — AD services](../screenshots/01-MIG-DC01-AD-Services.png) · [02 — DCDIAG output](../screenshots/02-MIG-DC01-DCDIAG-Health.png) · [03 — SYSVOL state](../screenshots/03-MIG-DC01-SYSVOL-Health.png) · [04 — Original FSMO owners](../screenshots/04-MIG-DC01-FSMO-Before-Migration.png)

### 3.2 System State backup

A dedicated **40 GB virtual disk** was attached to MIG-DC01, presented as NTFS volume **E:** with the label `AD-SystemState-Backup`. Windows Server Backup was used to perform a **System State** backup.

**Recorded result:** Backup completed successfully on October 31, 2026, between approximately **9:58 AM and 11:23 AM**, transferring **11.67 GB**.

**Recovery limitation:** The backup virtual disk remained on the same physical VirtualBox host; this was a lab rollback safeguard, **not** a geographically or physically independent disaster-recovery backup. A restore test was **not** documented.

**Evidence:** [05 — Backup VDI](../screenshots/05-MIG-DC01-Backup-Disk-Configuration.png) · [06 — Backup volume](../screenshots/06-MIG-DC01-Backup-Volume-Ready.png) · [07 — Successful backup](../screenshots/07-MIG-DC01-System-State-Backup-Success.png) · [08 — Backup details](../screenshots/08-MIG-DC01-System-State-Backup-Details.png)

## 4. Windows Server 2025 domain controller deployment

MIG-DC02 was configured as a Windows Server 2025 domain member with static IP `192.168.200.11`. The **Active Directory Domain Services** role was installed through Server Manager, followed by promotion as an **additional domain controller in the existing `migration.test` domain**. DNS Server and Global Catalog were included.

The promotion workflow included forest/schema and domain preparation steps. After promotion and restart, services and shares were checked:

```powershell
Get-Service -Name NTDS,DNS,KDC,Netlogon
net share
repadmin /replsummary
```

**Observed:** NTDS, DNS, KDC, and Netlogon were running; `SYSVOL` and `NETLOGON` shares were present; and the recorded replication summary showed **0 failures out of 5** in each direction between the two controllers. The `_ldap._tcp.dc._msdcs.migration.test` SRV query returned both controllers **before demotion**, as expected.

Schema verification used the AD schema object's `objectVersion` attribute; the resulting version was **91**.

**Evidence:** [09 — AD DS role](../screenshots/09-MIG-DC02-ADDS-Role-Installed.png) · [10 — Preparation options](../screenshots/10-MIG-DC02-Schema-Preparation-Options.png) · [11 — Services](../screenshots/11-MIG-DC02-AD-Services-Running.png) · [12 — SYSVOL/NETLOGON](../screenshots/12-MIG-DC02-SYSVOL-NETLOGON-Verified.png) · [13 — Replication summary](../screenshots/13-MIG-DC01-MIG-DC02-Replication-Success.png) · [14 — DNS SRV records](../screenshots/14-MIG-DC02-DNS-SRV-Records-Verified.png)

## 5. FSMO role transfer and time service

All five FSMO roles were transferred **gracefully through Microsoft management consoles**, rather than seized:

- **RID Master, PDC Emulator, Infrastructure Master:** Active Directory Users and Computers
- **Domain Naming Master:** Active Directory Domains and Trusts
- **Schema Master:** Active Directory Schema MMC snap-in

Verification:

```cmd
netdom query fsmo
```

**Observed final owners:**

```text
Schema master               MIG-DC02.migration.test
Domain naming master        MIG-DC02.migration.test
PDC                         MIG-DC02.migration.test
RID pool manager            MIG-DC02.migration.test
Infrastructure master       MIG-DC02.migration.test
```

Because MIG-DC02 became the PDC Emulator, its time service was reconfigured to use an external source in the lab:

```cmd
w32tm /config /manualpeerlist:"time.windows.com,0x8" /syncfromflags:manual /reliable:yes /update
```

```powershell
Restart-Service W32Time
```

```cmd
w32tm /resync /rediscover
w32tm /query /status
```

**Observed:** `Source: time.windows.com,0x8`, `Leap Indicator: 0 (no warning)`, and successful synchronization on October 31. This source was selected for the lab and should not be treated as a documented production time-service design.

**Evidence:** [15 — FSMO roles](../screenshots/15-MIG-DC02-All-FSMO-Roles-Transferred.png) · [16 — Time synchronization](../screenshots/16-MIG-DC02-NTP-Synchronization-Verified.png)

## 6. Client authentication and service continuity validation

MIG-CL01's preferred DNS server was changed from the old controller (`192.168.200.10`) to MIG-DC02 (`192.168.200.11`). During the period when both controllers were online, DC locator could still return MIG-DC01, which is normal in a multi-DC domain.

Relevant checks included:

```cmd
nltest /dsgetdc:migration.test /force
```

```powershell
Test-NetConnection MIG-DC02.migration.test -Port 389
```

The LDAP port test succeeded. To test independence from the original controller, MIG-DC01 was **temporarily powered off**. A subsequent forced DC locator query returned **MIG-DC02**, with PDC, GC, LDAP, KDC, and DNS flags.

A test account, **`adavis`**, then signed in to MIG-CL01 **for the first time** while MIG-DC01 was unavailable. This is a stronger validation than reusing a previously cached logon. A `klist` check showed a Kerberos TGT for `krbtgt/MIGRATION.TEST`, the KDC **MIG-DC02**, and **AES-256-CTS-HMAC-SHA1-96** encryption.

```cmd
klist
```

**Conclusion:** A live domain logon and Kerberos ticket issuance through the Windows Server 2025 controller were demonstrated while the original controller was unavailable. This is not a claim of full enterprise application compatibility testing.

**Evidence:** [17 — Kerberos authentication](../screenshots/17-MIG-DC02-Kerberos-Authentication-Verified.png)

## 7. Troubleshooting incident — DNS replication error 8524

### 7.1 Symptom

After the FSMO transfers and client authentication validation, a **normal (non-forced) demotion** of MIG-DC01 was attempted using the AD DS configuration wizard. The first attempt failed while transferring remaining data for:

```text
DC=ForestDnsZones,DC=migration,DC=test
```

The wizard reported that the directory service could not determine ownership of floating single-master operation roles. However, a separate `netdom query fsmo` check already showed all five roles on MIG-DC02. This made a straightforward “FSMO roles were not transferred” explanation inconsistent with the observed role ownership.

### 7.2 Investigation

The replication path was inspected directly:

```cmd
repadmin /showrepl MIG-DC01 "DC=ForestDnsZones,DC=migration,DC=test"
```

**Observed:** Inbound replication from MIG-DC02 reported **error 8524 — DNS lookup failure**.

The DNS client settings on MIG-DC01 were inspected:

```powershell
Get-DnsClientServerAddress -AddressFamily IPv4
```

**Observed:** MIG-DC01 still preferred its own DNS address, `192.168.200.10`, rather than the new controller's DNS address, `192.168.200.11`.

### 7.3 Corrective action

MIG-DC01's preferred DNS server was changed through the network adapter GUI to **192.168.200.11**. Replication of the affected naming context was then explicitly requested:

```cmd
repadmin /replicate MIG-DC01 MIG-DC02 "DC=ForestDnsZones,DC=migration,DC=test"
```

**Observed:** `Sync from MIG-DC02 to MIG-DC01 completed successfully.`

The normal demotion wizard was retried. **MIG-DC01 demoted successfully to a member server and restarted.** Forced demotion was not required.

### 7.4 Engineering interpretation

The directly observed fault was **DNS lookup failure during inbound `ForestDnsZones` replication**. Updating the old controller's DNS client setting, followed by successful replication, removed the blocker to graceful demotion. The sequence supports DNS configuration as the effective corrective action, without claiming every possible underlying cause of error 8524 was independently isolated.

**Operational lesson:** Before demoting an old controller, validate the DNS client configuration and inbound replication for each naming context, especially application partitions such as `ForestDnsZones` and `DomainDnsZones`. Successful FSMO transfer alone does not establish that demotion prerequisites are satisfied.

## 8. Post-demotion verification

### 8.1 Remaining domain controller and directory configuration

```powershell
Get-ADDomainController -Filter * |
    Select-Object HostName,OperatingSystem,IsGlobalCatalog

Get-ADDomainController -Identity MIG-DC02 |
    Select-Object HostName,IsGlobalCatalog,IsReadOnly
```

**Observed:** Only **MIG-DC02** was returned as a domain controller. It was a **Global Catalog** (`True`) and **not read-only** (`IsReadOnly=False`).

### 8.2 DNS locator records

```cmd
nslookup -type=SRV _ldap._tcp.dc._msdcs.migration.test 192.168.200.11
```

**Observed after demotion:** The SRV query returned **MIG-DC02 only**.

A DC GUID CNAME lookup against `192.168.200.11` also resolved successfully to `mig-dc02.migration.test`.

### 8.3 SYSVOL, directory diagnostics, and event review

The DFSR SYSVOL replicated folder on MIG-DC02 reported **State 4 (Normal)**. However:

```cmd
dcdiag /q
```

**Observed:** `DFSREvent` and `SystemLog` still reported warnings/errors. The event review included historical DFSR messages, KDC **Event ID 37** related to tickets constructed by the old controller, and DNS registration **Event ID 5774**. A later query for KDC Event ID 37 after **October 31, 2026, 20:10** returned **no matching events at the time of the check**; this is a bounded observation, not proof that such events could never recur.

**Important qualification:** Core domain services and authentication were validated, but a fully clean `dcdiag /q` result was **not** documented. Any claim of complete post-migration diagnostic remediation would require additional evidence.

## 9. Validation matrix

| Check | Result | Evidence / qualification |
|---|---|---|
| Original DC AD services running | **Pass** | Screenshot 01 |
| Original DC SYSVOL DFSR state | **Pass** | State 4; screenshot 03 |
| Original DC `dcdiag /q` | **Warnings** | DFSREvent/SystemLog; screenshot 02 |
| System State backup | **Pass** | Successful backup; screenshots 07–08; restore not tested |
| Windows Server 2025 AD DS promotion | **Pass** | Role, services, shares; screenshots 09–12 |
| Two-DC replication | **Pass at time checked** | `repadmin /replsummary` 0/5 failures each direction; screenshot 13 |
| AD schema version | **Verified** | Version 91 recorded |
| All five FSMO roles on MIG-DC02 | **Pass** | `netdom query fsmo`; screenshot 15 |
| PDC external time source | **Pass at time checked** | `w32tm /query /status`; screenshot 16 |
| First-time client logon while MIG-DC01 offline | **Pass** | `adavis` on MIG-CL01 |
| Kerberos ticket from MIG-DC02 | **Pass** | AES-256 TGT; screenshot 17 |
| Initial MIG-DC01 demotion | **Failed, diagnosed** | `ForestDnsZones`, error 8524 |
| Partition replication after DNS correction | **Pass** | Explicit `repadmin /replicate` succeeded |
| Normal demotion of MIG-DC01 | **Pass** | Completed and restarted |
| Only MIG-DC02 registered as DC | **Pass** | AD enumeration and DNS SRV lookup |
| Final SYSVOL DFSR state | **Pass** | State 4 on MIG-DC02 |
| Final `dcdiag /q` fully clean | **Not established** | Historical warnings/errors remained |
| MIG-CL02 post-migration DNS configuration | **Outstanding** | Not explicitly verified/updated |
| Backup restore test | **Not performed** | No restore evidence recorded |
| Domain/forest functional-level raise to 2025 | **Not verified** | No claim of upgrade |

## 10. AD object and organizational continuity

The domain contained the following test structure before migration:

- **OUs:** `Corporate-Users` (sub-OUs `IT`, `Finance`, `Human-Resources`), `Corporate-Groups`, and `Workstations`.
- **Users:** James Wilson (`jwilson`), Sarah Mitchell (`smitchell`), and Angela Davis (`adavis`).
- **Security groups:** `SG-IT-Users`, `SG-Finance-Users`, and `SG-HR-Users`, each with its corresponding user.
- **Computer objects:** `MIG-CL01` and `MIG-CL02` in the `Workstations` OU.

The recorded session included validation of users, groups, and memberships on the new controller. The successful `adavis` sign-in additionally demonstrated that a migrated directory identity could authenticate against MIG-DC02.

## 11. Outstanding items and recommendations

1. **Verify MIG-CL02 DNS:** Confirm its preferred DNS server points to `192.168.200.11`, not the retired controller at `.10`, and validate DC locator and logon.
2. **Review residual event logs:** Recheck DFSR, KDC, and DNS registration events after a fresh monitoring window. Distinguish historical events from active failures before asserting a clean `dcdiag` result.
3. **Reconfirm DNS hygiene:** Verify no stale records or client settings reference MIG-DC01 as a domain controller or DNS server. Review the VirtualBox DHCP-provided DNS options as applicable.
4. **Document backup recovery separately:** If expanding this into a disaster-recovery exercise, test restore procedures in a safely isolated copy; use physically independent backup storage for a production design.
5. **Functional level:** Treat any later raise of domain/forest functional levels as a separate, planned, compatibility-checked change. No such change is claimed here.

These items are **follow-up validation or enhancement work**, not evidence that the documented migration failed.

## 12. Lessons learned

- **Side-by-side migration reduces risk:** A second controller can be promoted and validated before the original controller is retired.
- **FSMO transfer is necessary but not sufficient:** Demotion can still be blocked by replication of a directory naming context.
- **DNS is foundational to AD operations:** A name-resolution failure can surface as a replication or demotion problem.
- **Test live authentication, not only cached logon:** First-time user sign-in while the old DC was offline provided persuasive continuity evidence.
- **Preserve diagnostic nuance:** A successful migration can coexist with historical or unresolved event-log findings; document both.
- **Recovery readiness has limits:** A successful System State backup is valuable, but same-host storage and lack of a restore test are meaningful constraints.

---

## 13. Evidence index

All 17 screenshots are stored in the repository's [`screenshots/`](../screenshots/) directory:

| # | Evidence |
|---|---|
| 01 | [MIG-DC01 AD services](../screenshots/01-MIG-DC01-AD-Services.png) |
| 02 | [MIG-DC01 DCDIAG findings](../screenshots/02-MIG-DC01-DCDIAG-Health.png) |
| 03 | [MIG-DC01 SYSVOL health](../screenshots/03-MIG-DC01-SYSVOL-Health.png) |
| 04 | [FSMO owners before migration](../screenshots/04-MIG-DC01-FSMO-Before-Migration.png) |
| 05 | [Backup disk configuration](../screenshots/05-MIG-DC01-Backup-Disk-Configuration.png) |
| 06 | [Backup volume ready](../screenshots/06-MIG-DC01-Backup-Volume-Ready.png) |
| 07 | [System State backup success](../screenshots/07-MIG-DC01-System-State-Backup-Success.png) |
| 08 | [System State backup details](../screenshots/08-MIG-DC01-System-State-Backup-Details.png) |
| 09 | [MIG-DC02 AD DS role installed](../screenshots/09-MIG-DC02-ADDS-Role-Installed.png) |
| 10 | [Schema preparation options](../screenshots/10-MIG-DC02-Schema-Preparation-Options.png) |
| 11 | [MIG-DC02 AD services running](../screenshots/11-MIG-DC02-AD-Services-Running.png) |
| 12 | [MIG-DC02 SYSVOL and NETLOGON](../screenshots/12-MIG-DC02-SYSVOL-NETLOGON-Verified.png) |
| 13 | [Replication summary](../screenshots/13-MIG-DC01-MIG-DC02-Replication-Success.png) |
| 14 | [DNS SRV records verified](../screenshots/14-MIG-DC02-DNS-SRV-Records-Verified.png) |
| 15 | [All FSMO roles transferred](../screenshots/15-MIG-DC02-All-FSMO-Roles-Transferred.png) |
| 16 | [NTP synchronization](../screenshots/16-MIG-DC02-NTP-Synchronization-Verified.png) |
| 17 | [Kerberos authentication against MIG-DC02](../screenshots/17-MIG-DC02-Kerberos-Authentication-Verified.png) |

**Project status:** Side-by-side AD DS migration and graceful retirement of the original domain controller **completed**; follow-up validation items remain documented above.

