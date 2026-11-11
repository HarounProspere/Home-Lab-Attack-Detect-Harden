# Home Lab: Attack, Detect, Harden

**Author:** Haroun Prospere
**Date:** November 2026
**Environment:** Two virtual machines on an Apple Silicon Mac (UTM)

## Overview

I built a small private network with two virtual machines, then ran a real
attack against one of them, detected it in the logs, and hardened the system
so the same attack failed. The project walks through a full defensive cycle:
build, attack, detect, and harden. It maps to the Security Operations and
Security Architecture domains of CompTIA Security+.

- **Target:** Ubuntu Server running SSH and an Nginx web server, protected by a UFW firewall
- **Attacker:** Kali Linux, used to scan and attack the target

## Tools and Technologies

- Ubuntu Linux (target) and Kali Linux (attacker)
- UTM virtualization (Apple Silicon)
- Nmap (network scanning)
- Hydra (brute-force attack)
- Wireshark / system logs (detection)
- UFW firewall, fail2ban, key-only SSH (hardening)

## 1. Build the Target

Set up an Ubuntu server with two services exposed, giving it a realistic
attack surface, and confirmed each was running.

```
sudo apt install openssh-server nginx ufw -y
sudo systemctl enable --now ssh
sudo systemctl enable --now nginx
```

I learned the difference between a service being *installed* and being
*active*: SSH was installed but inactive until I explicitly enabled it.

## 2. Configure the Firewall

Checked the firewall's starting state (inactive), allowed only the two ports
I wanted reachable, then enabled it. Allowing SSH *before* enabling the
firewall is deliberate, so I would not lock myself out.

```
sudo ufw allow ssh
sudo ufw allow 80/tcp
sudo ufw enable
```

Result: only ports 22 (SSH) and 80 (HTTP) are allowed; everything else is
denied. This is attack surface reduction.

## 3. Reconnaissance (Nmap)

From Kali, scanned the target to find open services.

```
nmap -sV 192.168.64.6
```

Findings:
- 22/tcp open, OpenSSH
- 80/tcp open, Nginx
- 998 ports filtered (blocked by the firewall)

The filtered ports confirm the firewall is working. The two open ports are
my attack surface. An open port is not automatically a weakness; it only
becomes one if the service behind it is weak.

## 4. Brute-Force Attack (Hydra)

To demonstrate a realistic attack, I created a test account with a weak
password, then attacked SSH from Kali.

```
hydra -l testuser -P words.txt ssh://192.168.64.6
```

Result: Hydra found the password in about four seconds. This is why SSH
password login is a weak default. (First attempt failed due to a username
typo, which I found and corrected, a good reminder to verify setup before
blaming the tool.)

## 5. Detection (Logs)

Found the attack on the target by searching the SSH logs.

```
sudo journalctl -u ssh | grep Failed
```

Every failed login came from the same IP (Kali, 192.168.64.7), and several
hit within the same second. Many failures in a short time from one address
is the signature of a brute-force attack, not a user who forgot a password.

## 6. Hardening

Applied three layers of defense:

**fail2ban** automatically bans an IP after repeated failures.

```
sudo apt install fail2ban -y
```

A test run with wrong passwords triggered a ban of Kali's IP, confirmed with:

```
sudo fail2ban-client status sshd
```

**Key-only SSH** turns off password login entirely. I generated a key on
Kali, copied it to the server, then disabled passwords and root login.

```
ssh-keygen -t ed25519
ssh-copy-id harounpros@192.168.64.6
# on the server:
PasswordAuthentication no
PermitRootLogin no
```

## 7. Verify (Before and After)

Ran the exact same Hydra attack again.

```
hydra -l testuser -P words.txt ssh://192.168.64.6
```

Result: `[ERROR] target does not support password authentication`.

The attack that cracked the password minutes earlier could no longer even
attempt it. The hardening worked.

## Results

- Reduced the attack surface to two necessary ports
- Detected a live brute-force attack in the logs
- Automatically banned the attacker with fail2ban
- Eliminated password guessing by switching to key-only SSH
- Proved the fix by re-running the same attack and watching it fail

## Skills Demonstrated

- Linux command-line administration
- Network scanning and service enumeration
- Log analysis and attack detection
- Firewall configuration and attack surface reduction
- Defense in depth and system hardening
- Verifying security controls rather than assuming them

## Screenshots

See the `/screenshots` folder for evidence of each stage: services running,
firewall before/after, Nmap scan, Hydra cracking the password, failed logins
in the logs, fail2ban banning the attacker, and the final attack failing.
