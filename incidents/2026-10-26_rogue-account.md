# IR-2026-003 — Rogue admin account created on ubuntu-victim

**Analyst:** Ivan Bildzia
**Date:** 2026-10-26 (times in UTC)
**Detected by:** Wazuh rules 5902 / 5901 / 5402 / 5555
**MITRE:** T1136.001 Create Account · T1098 Account Manipulation · T1548 Abuse Elevation Control
**Status:** closed — rogue account removed

## Scenario
This simulates what an attacker does *after* getting a foothold: create a
hidden account and give it root, so they can get back in later even if the
original hole is closed (persistence + privilege escalation). I ran the actions
on ubuntu-victim and then investigated them from the SIEM side.

Actions performed (as the compromised `lil-mixail` account, via sudo):
```bash
useradd -m -s /bin/bash backdoor
echo 'backdoor:Pwned123!' | chpasswd
usermod -aG sudo backdoor
echo 'backdoor ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/backdoor
```

## What Wazuh detected
| Rule | Level | Description |
|---|---|---|
| 5902 | 8 | New user added to the system |
| 5901 | 8 | New group added to the system |
| 5555 | 3 | PAM: user changed password |
| 5402 | 3 | Successful sudo to ROOT executed |

Raw evidence from the logs:
```
Oct 26 19:15:08 ubuntu-victim sudo[7244]: lil-mixail : ... USER=root ;
    COMMAND=/usr/sbin/useradd -m -s /bin/bash backdoor
Oct 26 19:15:09 ubuntu-victim useradd[7246]: new group: name=backdoor, GID=1001
Oct 26 19:15:09 ubuntu-victim useradd[7246]: new user: name=backdoor, UID=1001,
    home=/home/backdoor, shell=/bin/bash
Oct 26 19:15:14 ubuntu-victim chpasswd[7257]: password changed for backdoor
Oct 26 19:15:19 ubuntu-victim sudo[7260]: ... COMMAND=/usr/sbin/usermod -aG sudo backdoor
Oct 26 19:15:23 ubuntu-victim sudo[7270]: ... COMMAND=/usr/bin/tee /etc/sudoers.d/backdoor
```

## Investigation notes
- The `sudo` logs show **who** did it: the `lil-mixail` account, escalating to
  root. In a real case that account would be treated as compromised and the
  investigation would start there.
- The new-user event (rule 5902, level 8) is the strongest signal. That's the
  one I'd want an alert on.

## Detection gap I noticed
The actual privilege grant — `usermod -aG sudo` and writing
`/etc/sudoers.d/backdoor` — did **not** raise a dedicated high-level alert; it
only appeared as a generic "sudo to root" (level 3). So an attacker editing
sudoers directly would be easy to miss. Wazuh's own SCA check 35731 flagged the
same weakness: audit rules for `/etc/sudoers.d` aren't configured
(`Could not open dir '/etc/audit/rules.d'`).

**Fix planned:** add File Integrity Monitoring on `/etc/sudoers.d/` so any change
there generates its own alert (done in the next step).

## Impact
In the lab: a working root-level backdoor account existed until removed. In a
real environment this would be a full compromise of the host.

## Response / Remediation
Removed the rogue account and its sudoers file:
```bash
sudo userdel -r backdoor
sudo rm -f /etc/sudoers.d/backdoor
```
Then verified it was gone (`id backdoor` returns "no such user").

## Recommendations
- Alert on new local accounts, especially additions to the `sudo`/`admin` group.
- Add FIM on `/etc/sudoers`, `/etc/sudoers.d/`, `/etc/passwd`, `/etc/shadow`.
- Enable the Linux audit rules the CIS benchmark asks for (scope changes).
- Review why `lil-mixail` could be used this way — enforce MFA, limit sudo.

---
*Lab only — all of this was done against my own VM.*
