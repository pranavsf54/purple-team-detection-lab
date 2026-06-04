# Domain Controller (DC01) build notes

## Domain

- Forest: corp.lab.lan
- NetBIOS: CORP
- Functional level: Windows Server 2016
- DC: DC01.corp.lab.lan at 192.168.10.10

## Population

Populated by BadBlood (https://github.com/davidprowe/BadBlood), which created
approximately 2,500 users, hundreds of groups, and randomized ACLs across
the default Users container and several BadBlood-generated OUs.

## Seeded misconfigurations for the Phase 4 attack chain

### 1. Kerberoastable service account
- Username: `sqlsvc` (CN under IT OU)
- Password: `Summer2024!` (present in rockyou wordlist)
- SPN: `MSSQLSvc/sqlserver.corp.lab.lan:1433`
- Attack: T1558.003 Kerberoasting
### 2. AS-REP roastable account
- Username: `jsmith` (CN under HR OU)
- Password: `Welcome123!` (present in rockyou)
- Account option: "Do not require Kerberos preauthentication" enabled
- Attack: T1558.004 AS-REP Roasting

### 3. GPP cleartext password
- GPO: `Mapped Drive Setup` (linked at domain root)
- Stored credential: `corp\backup-svc` / `BackupP@ss123`
- Location: SYSVOL\corp.lab.lan\Policies\{GUID}\User\Preferences\Drives\Drives.xml
- Attack: T1552.006 GPP Password / NetExec gpp-password module

### 4. Weak ACL: GenericWrite
- Attacker user: HOLLIE_MANN
- Target user: BUDDY_MCCARTY
- Permission: GenericWrite on target object
- Attack: T1098 Account Manipulation / ACL abuse via BloodHound

## Custom OUs

Workstations, Servers, IT, HR, Finance, Executives
