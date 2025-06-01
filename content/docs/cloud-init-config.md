---
title: "cloud-init config"
weight: 1
# bookFlatSection: false
# bookToc: true
# bookHidden: false
# bookCollapseSection: false
# bookComments: false
# bookSearchExclude: false
---

## Scope

This page list the various cloud-init config file I'm using when creating a new server. Unless specified, I'm working with a Debian Bookworm install, on an ARM CPU, using [Hetzner](https://www.hetzner.com/) as a cloud provider.

You can also check [cloud-init official site](https://cloud-init.io/) and [cloud config examples](https://cloudinit.readthedocs.io/en/latest/reference/examples.html).

## Minimal cloud-init config

This minimal config files sets up a user, gives it `sudo` privileges, sets up fail2ban and prevent SSH for root and without a SSH key.

Replace the content of the highlighted lines as specified.

```yml {hl_lines=[2,4,5,7] style=emacs}
#cloud-config
timezone: YOUR_TIMEZONE eg. Europe/London 
users:
  - name: YOUR_USERNAME
    passwd: OUTPUT_FROM mkpasswd -m sha-512
    ssh_authorized_keys:
      - YOUR_PUBLIC_KEY
    groups: sudo
    shell: /bin/bash
    lock_passwd: false
packages:
  - fail2ban
  - python3-systemd
package_update: true
package upgrade: true
write_files:
- content: |
    [sshd]
    backend = systemd
    enabled = true
    banaction = iptables-multiport
  path: /etc/fail2ban/jail.local
runcmd:
  - service fail2ban enable
  - sed -i -r 's/^#?PermitRootLogin.*$/PermitRootLogin no/' /etc/ssh/sshd_config
  - sed -i -r 's/^#?PasswordAuthentication.*$/PasswordAuthentication no/' /etc/ssh/sshd_config 
  - sed -i -r 's/^#?PermitEmptyPasswords.*$/PermitEmptyPasswords no/' /etc/ssh/sshd_config 
  - sed -i -r 's/^#?PubkeyAuthentication.*$/PubkeyAuthentication yes/' /etc/ssh/sshd_config  
  - sed -i -r 's/^#?StrictModes.*$/StrictModes yes/' /etc/ssh/sshd_config 
  - sed -i -r 's/^#?MaxAuthTries.*$/MaxAuthTries 2/' /etc/ssh/sshd_config 
  - sed -i -r 's/^#?StrictModes.*$/StrictModes yes/' /etc/ssh/sshd_config 
  - sed -i -r 's/^#?UsePAM.*$/UsePAM no/' /etc/ssh/sshd_config  
  - sed -i -r 's/^#?X11Forwarding.*$/X11Forwarding no/' /etc/ssh/sshd_config    
  - sed -i -r 's/^#?AllowAgentForwarding.*$/AllowAgentForwarding no/' /etc/ssh/sshd_config    
  - sed -i -r 's/^#?AllowTcpForwarding.*$/AllowTcpForwarding no/' /etc/ssh/sshd_config    
  - reboot
```