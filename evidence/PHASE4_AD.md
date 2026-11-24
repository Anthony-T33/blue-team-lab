# Phase 4 — Active Directory Hardening (Samba AD DC)

## Objective
Build a working Active Directory domain on the lab and apply + verify core
identity-hardening controls. This backs the "Active Directory hardening" CV claim
with a real, reproducible environment.

## Why Samba AD DC
The lab runs on Apple Silicon (ARM64). Microsoft does not freely distribute a
Windows Server **ARM64** evaluation ISO (it is Azure/OEM-oriented), so a classic
Windows Server DC is not practical here. **Samba 4** implements the same Active
Directory protocols (LDAP, Kerberos, NTLM, DNS, GPO schema) and runs natively on
ARM64 — a legitimate, fully-functional AD domain to harden.

## Architecture
| Component | Detail |
|---|---|
| VM | **DC01** (Ubuntu 24.04 Server, ARM64) |
| IP | 192.168.64.8 |
| Resources | 2 GB RAM / 2 vCPU / 20 GB disk |
| Role | Active Directory Domain Controller |
| Software | Samba 4.19.5 (`samba-tool`), SAMBA_INTERNAL DNS |
| Domain | `lab.local` (NetBIOS `LAB`) |

## Provisioning
```bash
hostnamectl set-hostname dc01.lab.local
apt-get install -y samba smbclient winbind krb5-config krb5-user ldb-tools
rm -f /etc/samba/smb.conf   # remove the "standalone server" default
samba-tool domain provision --use-rfc2307 --realm=LAB.LOCAL --domain=LAB \
    --adminpass='<redacted>' --server-role=dc --dns-backend=SAMBA_INTERNAL
systemctl enable --now samba-ad-dc
```

## Hardening applied (password & account policy)
| Control | Default | Hardened |
|---|---|---|
| Password complexity | on | on |
| Minimum password length | 7 | **14** |
| Maximum password age (days) | 42 | **60** |
| Password history length | 24 | 24 |
| Account lockout threshold | 0 (disabled) | **5** |
| Account lockout duration (min) | 30 | 30 |

Commands:
```bash
samba-tool domain passwordsettings set --complexity=on --history-length=24 \
    --min-pwd-length=14 --min-pwd-age=1 --max-pwd-age=60
samba-tool domain passwordsettings set --account-lockout-duration=30 \
    --account-lockout-threshold=5 --reset-account-lockout-after=30
```

**Proof the policy is enforced:** creating a user with a 13-char password is
rejected (`check_password_restrictions: the password is too short ... equal or
longer than 14`); a 15-char password is accepted.

## Identity objects
- Users: `Administrator`, `alice`, `bob`
- Group: `SOC Analysts` (members: alice, bob)

## Verification
- Forward DNS: `dc01.lab.local -> 192.168.64.8` (Samba internal DNS)
- Kerberos: `kinit alice@LAB.LOCAL` issues a TGT from `krbtgt/LAB.LOCAL@LAB.LOCAL`
- Password expiry reflects the 60-day max-age policy

## Next
- Join a Windows 11 ARM64 client to `LAB.LOCAL`, deploy Sysmon, stream to the SIEM.
