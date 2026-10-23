---
layout: post
title: "Essential Linux Security Hardening for Production DevOps Servers"
date: 2026-10-20 09:00:00 +0530
author: Chandana Nawarathna
categories: [DevOps]
tags: [linux, security, hardening, devops, production]
description: "Practical security hardening measures for Linux servers running DevOps tools and workloads in production."
---

# Essential Linux Security Hardening for Production DevOps Servers

When running DevOps tools and workloads on Linux servers in production, security hardening is not optional — it's a requirement. This article covers practical, non-destructive measures you can apply to reduce the attack surface of your infrastructure.

## Why Harden Your DevOps Servers?

DevOps servers often hold sensitive data: SSH keys, container registry credentials, CI/CD pipeline configurations, and infrastructure state. A compromised server can lead to full infrastructure takeover, supply chain attacks, or data exfiltration.

Hardening reduces the risk of common attack vectors while maintaining operational usability.

## 1. Keep the System Updated

Regular updates are the first line of defense. Set up automatic security updates or establish a patching cadence.

```bash
# Debian/Ubuntu
sudo apt update && sudo apt upgrade -y

# RHEL/CentOS/Fedora
sudo dnf upgrade --security -y
```

Consider enabling unattended upgrades on Debian-based systems:

```bash
sudo dpkg-reconfigure unattended-upgrades
```

## 2. Harden SSH Configuration

SSH is the primary attack vector for server compromise. Apply these changes in `/etc/ssh/sshd_config`:

```bash
# Disable root login
PermitRootLogin no

# Disable password authentication (use key-based auth instead)
PasswordAuthentication no

# Change the default listening port (security through obscurity, but reduces automated attacks)
Port 2222

# Limit user access
AllowUsers devops-team

# Set idle timeout
ClientAliveInterval 300
ClientAliveCountMax 0
```

After changes, reload SSH:

```bash
sudo systemctl reload sshd
# or
sudo systemctl restart sshd
```

**Never disconnect SSH before verifying the new configuration works from another session.**

## 3. Apply Kernel Hardening with sysctl

Linux kernel parameters can significantly reduce exploit impact. Create `/etc/sysctl.d/99-hardening.conf`:

```conf
# Prevent privileged port reuse
net.ipv4.ip_unprivileged_port_start=80

# Ignore ICMP echo requests (stealth mode)
net.ipv4.ic_echo_ignore_all=1

# Enable source path validation
net.ipv4.conf.all.rp_filter=1
net.ipv4.conf.default.rp_filter=1

# Disable IP source routing
net.ipv4.conf.all.accept_source_route=0
net.ipv4.conf.default.accept_source_route=0

# Enable syncookies to handle SYN flood attacks
net.ipv4.tcp_syncookies=1
```

Apply immediately:

```bash
sudo sysctl --system
```

## 4. Restrict Network Exposure

Only expose necessary ports. Use a firewall to enforce boundaries.

```bash
# Debian/Ubuntu with UFW
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 203.0.113.0/24 to any port 22
sudo ufw enable

# RHEL/CentOS with firewalld
sudo firewall-cmd --set-default-zone=drop
sudo firewall-cmd --add-port=22/tcp --zone=trusted
sudo firewall-cmd --reload
```

## 5. Minimize Installed Packages

Install only what is needed. Remove unused software and dependencies.

```bash
# Debian/Ubuntu
sudo apt autoremove --purge $(dpkg -l | grep '^rc' | awk '{print $2}')

# RHEL/CentOS
sudo dnf autoremove
sudo package-cleanup --oldkernels --count=2
```

## 6. Secure File and Directory Permissions

Follow the principle of least privilege. Key directories and their recommended permissions:

| Directory         | Recommended Permission | Notes                      |
|-------------------|------------------------|----------------------------|
| /etc              | 755 (root:root)        | World-readable is acceptable |
| /var/www          | 755 or tighter         | Adjust for your web roots  |
| /home             | 711 or 750             | Prevent directory listing   |
| /opt              | 755                    | Standard for add-on software |
| /tmp              | 1777 (sticky bit)      | Prevents file overwrites    |

Use `chattr +i` on critical system binaries and configuration files to make them immutable.

## 7. Enable Audit Logging

Audit logs help you investigate incidents. Enable Linux auditing:

```bash
sudo apt install auditd auditui -y  # Debian/Ubuntu
sudo dnf install audit -y             # RHEL/CentOS
sudo systemctl enable auditd --now
```

Track changes to critical files:

```bash
sudo auditctl -w /etc/sudoers -p wa -k sudoers-change
sudo auditctl -w /etc/ssh/sshd_config -p wa -k sshd-change
```

View logs with:

```bash
sudo ausearch -k sudoers-change
```

## 8. Isolate Workloads with Namespaces and cgroups

When running custom services, use Linux namespaces and cgroups for isolation:

```bash
# Example: running a service with resource limits
systemd-run --unit=my-service \
  --property=MemoryMax=512M \
  --property=CPUQuota=1.0 \
  /opt/my-service/run.sh
```

## 9. Monitor for Changes

Hardening is not a set-and-forget task. Use file integrity monitoring:

```bash
# Install and configure AIDE (Advanced Intrusion Detection Environment)
sudo apt install aide -y
sudo aideinit --init
sudo cp /var/lib/aide/aide.db.new /var/lib/aide/aide.db
sudo systemctl enable aidew
```

Integrate with your monitoring pipeline (Prometheus + Alertmanager, Datadog, etc.).

## 10. Backup and Recovery

Hardening means nothing without backups. Follow the 3-2-1 rule:

- 3 copies of data
- 2 different media types
- 1 offsite copy

Test restoration regularly. Infrastructure as Code (Terraform, Ansible) makes recovery reproducible.

## Conclusion

Security hardening is a layered practice. No single measure makes a server impenetrable, but combined these practices raise the cost of an attack and reduce risk. Start with the basics (updates, SSH, firewall) and incrementally add more layers as your comfort and requirements grow.

*This article is intended for educational and operational purposes. Always test changes in a staging environment before applying to production, and consult your organization's security policies.*