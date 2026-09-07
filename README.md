# Ansible-linux-baseline

A minimal, opinionated Ansible playbook for hardening a fresh ubuntu/rocky server. Run it once and the server comes out with SSH locked down, a default deny firewall, brute-force protection, tailscale mesh VPN and zram, creating a baseline to run anything on it.

![Ansible](https://img.shields.io/badge/Ansible-EE0000?style=flat-square&logo=ansible&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-E95420?style=flat-square&logo=ubuntu&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-green?style=flat-square)
![Security Hardened](https://img.shields.io/badge/Security-Hardened-brightgreen?style=flat-square&logo=linuxfoundation&logoColor=white)
![Tailscale](https://img.shields.io/badge/Tailscale-Integrated-5B49E9?style=flat-square&logo=tailscale&logoColor=white)
![Firewall](https://img.shields.io/badge/Firewall-UFW%20%2B%20Fail2Ban-blue?style=flat-square&logo=shautomatik&logoColor=white)
![CI](https://github.com/Tanvir101cmd/ansible-linux-baseline/actions/workflows/test-playbook.yml/badge.svg)
![CI](https://github.com/Tanvir101cmd/ansible-linux-baseline/actions/workflows/molecule.yml/badge.svg)

## Table of Contents

- [Core Features](#core-features)
- [Repository Structure](#repository-structure)
- [Usage](#usage)
  - [Prerequisites](#prerequisites)
  - [Server Pre-configuration](#1-server-pre-configuration)
  - [Install Ansible and Collections](#2-install-ansible-and-collections)
  - [Create Inventory File](#3-create-inventory-file)
  - [Configure Variables](#4-configure-your-variables)
  - [Run the Playbook](#5-run-the-ansible-playbook)
- [Troubleshooting](#troubleshooting)
- [Testing](#testing)
- [Roadmap](#roadmap)
- [References and Acknowledgments](#references-and-acknowledgments)
- [License](#license)

## Core Features

After a successful run the server will have:

- SSH restricted to key-based authentication only on port `2222`, root login disabled
- UFW enabled with default-deny incoming, and rate limited SSH port
- Fail2ban automatically banning after 5 failed attempts for 24 hour
- Tailscale installed and running for remote access
- Automatic security updates applied via unattended-upgrades
- Zram swap active, default `swap.img` removed

---

## Repository Structure

```bash
.
├── ansible.cfg
├── CHANGELOG.md
├── docs
│   └── MANUAL_SETUP.md
├── generate-changelog.py
├── LICENSE
├── molecule
│   ├── rocky
│   │   ├── converge.yml
│   │   ├── molecule.yml
│   │   └── verify.yml
│   └── ubuntu
│       ├── converge.yml
│       ├── molecule.yml
│       └── verify.yml
├── playbook.yml
├── README.md
└── roles
    └── linux_baseline
        ├── handlers
        │   └── main.yml
        ├── tasks
        │   ├── base.yml
        │   ├── fail2ban.yml
        │   ├── lynis.yml
        │   ├── main.yml
        │   ├── security_firewalld.yml
        │   ├── security_ufw.yml
        │   ├── security.yml
        │   ├── ssh.yml
        │   ├── system.yml
        │   └── tailscale.yml
        └── vars
            ├── debian.yml
            ├── main.yml
            └── redhat.yml
```

## Usage

### Prerequisites

- Server running **Ubuntu 22.04+ or Rocky Linux 10+**
- A user with `sudo` access on the target server
- SSH access to the target server from the host machine
- Ansible installed on **host machine** (the machine you run the playbook from)
- `ansible.posix` and `community.general` collections

### 1. Server Pre-configuration

To ensure maximum security, we will manually harden ssh to disable password authentication *before* granting Ansible its required deployment privileges. 

Follow this exact sequence:

#### Step 0: Make a ssh key (if needed)

If you do not already have an SSH key pair on your **local machine**, generate a modern, secure ed25519 key:

```bash
ssh-keygen -t ed25519
```

#### Step 1: Push your SSH Public Key to the Server

From your **local machine**, push your public key file to the remote server. Replace `your_username` and `192.168.0.150` with your actual setup:

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub your_username@192.168.0.150
```

#### Step 2: SSH into the Server to Verify

Verify that your local machine can use the key to establish an encrypted session:

```bash
ssh -i ~/.ssh/id_ed25519 your_username@192.168.0.150
```

#### Step 3: Disable Password Logins

Once inside the remote server terminal, turn off password-based authentication permanently and restart the SSH daemon:

```bash
sudo sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config

# If you are on ubuntu, please also delete the 50-cloud-init.conf as well to let it not override the sshd_config
sudo rm /etc/ssh/sshd_config.d/50-cloud-init.conf

# Finally restart the ssh or sshd for rhel 
sudo systemctl restart ssh ; sudo systemctl restart sshd 
```

#### Step 4: Test login via ssh key

**DO NOT close your current terminal window yet!** If you made a typo, you will be locked out of the server.

Open a brand new, separate terminal window on your local machine and attempt to log in:

```bash
ssh -i ~/.ssh/id_ed25519 your_username@192.168.0.150
```

- **If it works without asking for password:** Your ssh lockdown is working perfectly. Now the only way to get into that server is that ssh-key
- **If it fails:** Use your first, still-open terminal window to debug your `/etc/ssh/sshd_config` file

#### Step 5: Configure Passwordless Sudo

Now that the server is completely locked down to physical SSH keys only, it is entirely safe to grant passwordless sudo privileges so Ansible can run its automation without hitting interactive TTY prompt blocks. Be sure to use **`sudo visudo`** instead of **`sudo nano`**

Type:

```bash
sudo visudo -f /etc/sudoers.d/your_username-ansible
```

and paste the following rule:

```ini
your_username ALL=(ALL) NOPASSWD: ALL
```

---

### 2. Install ansible and collections

Install Ansible on your host machine:

```bash
# Ubuntu / Debian
sudo apt update && sudo apt install ansible -y

# Fedora / RHEL
sudo dnf install ansible -y

# Arch Linux
sudo pacman -S --noconfirm ansible

# macOS
brew install ansible
```

Then install the required collections:

```bash
ansible-galaxy collection install ansible.posix community.general
```

---

### 3. Create inventory file

Create a hosts.ini file in the project root:

`hosts.ini`

```ini
[homelab]
192.168.0.150 ansible_port=22 ansible_user=your_username ansible_ssh_private_key_file=~/.ssh/id_ed25519
```

---

### 4. Configure your variables

Open `roles/linux_baseline/vars/main.yml` and set your values:

```yaml
linux_baseline_username: "your_username"             # Primary user on the server
linux_baseline_pub_key: "~/.ssh/id_ed25519.pub"      # Path to ssh public key
linux_baseline_ssh_port: "2222"                      # Set your custom ssh port
```

> Note: If linux_baseline_username is left as your_username, the user-creation and key-deploy steps are skipped

---

### 5. Run the Ansible playbook

Execute the playbook with the following command:

```bash
ansible-playbook -i hosts.ini playbook.yml
```

 Or run specific sections:

| Tag       | What it does                   |
| -----------| --------------------------------|
| packages  | System update + base packages  |
| ssh       | SSH hardening + key deployment |
| security  | ufw/firewalld + fail2ban       |
| tailscale | Mesh VPN service               |
| lynis     | Lynis audit suggestions        |
| system    | zram + swapfile removal        |

``` bash
# Security hardening only
ansible-playbook -i hosts.ini playbook.yml --tags security

# SSH setup only
ansible-playbook -i hosts.ini playbook.yml --tags ssh

# Everything except the system
ansible-playbook -i hosts.ini playbook.yml --skip-tags system
```

---

## Troubleshooting

- ### **Playbook hangs at the start**

  This is a TTY timeout issue. Make sure to ran the sudoers pre-configuration step on the target server before running the playbook.


- ### **UFW locked me out of SSH**

  If the playbook fails mid-run and UFW is left in a broken state, access your server via your hosting provider's console and run `sudo ufw disable` to recover access, then re-run the playbook from the beginning.


- ### **Tailscale task fails**

  The Tailscale installer requires internet access from the target server. If your server is behind a restrictive firewall, allow outbound traffic on port `443` before running.

### For manual step-by-step setup without Ansible, see [Manual Setup](./docs/MANUAL_SETUP.md).

---

## Testing

This playbook is tested using [Molecule](https://ansible.readthedocs.io/projects/molecule/) with podman as the container driver. Tests run automatically on every push via GitHub Actions.

[![Molecule CI](https://github.com/Tanvir101cmd/ansible-linux-baseline/actions/workflows/molecule.yml/badge.svg)](https://github.com/Tanvir101cmd/ansible-linux-baseline/actions/workflows/molecule.yml)

### Test matrix

| Distro         | Status |
| ----------------| --------|
| Debian 12      | ✅      |
| Ubuntu 22.04   | ✅      |
| Rocky Linux 10 | ✅      |


---

## Roadmap

- [ ] NTP hardening
- [ ] SSH banner / MOTD
- [x] Molecule tests for playbook validation
- [x] Distro-agnostic support (Debian, Rocky Linux)

### For a full list of changes, see [CHANGELOG.md](./CHANGELOG.md).

---

## References and Acknowledgments

### [Lynis by Cisofy](https://github.com/cisofy/lynis) - The open-source security auditing tool to benchmark and guide these hardening configurations

### [Linux Audit Blog](https://linux-audit.com) - A valuable resource website for in-depth technical guide for standard Linux system hardening practices.  

---

## License

This project is licensed under the **MIT License** - see the [LICENSE](./LICENSE) file for details.

Copyright (c) 2026 [@Tanvir101cmd](https://github.com/Tanvir101cmd)
