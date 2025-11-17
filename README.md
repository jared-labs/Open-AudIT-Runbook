# VMINV01 — Open-AudIT Community Install Runbook

Deploy an Ubuntu Server 24.04 VM on Proxmox for network discovery and asset inventory using **Open-AudIT Community Edition**. This runbook captures the real-world quirks we hit with the `.run` installer, Apache config, and 403 errors. Copy/paste friendly.

> Role: Dedicated inventory scanner for the homelab  
> Stack: Ubuntu 24.04 + Apache + PHP + Open-AudIT Community  
> Access: `http://<VMINV01_IP>/open-audit/` (first login: `admin` / `password` — change immediately)

* * *

## At a Glance

- **VM name (Proxmox):** `VMINV01`
- **Hostname (inside VM):** `vminv01`
- **OS:** Ubuntu Server 24.04 (minimal)
- **Purpose:** Network discovery + asset inventory (Open-AudIT Community)
- **Web server:** Apache 2
- **PHP:** Ubuntu default (via `php` + `libapache2-mod-php`)
- **Open-AudIT install path:** `/usr/local/open-audit`
- **Web path:** `http://<VMINV01_IP>/open-audit/`
- **First login:**
  - Username: `admin`
  - Password: `password` (change on first login)
- **Key Apache modules:**
  - `rewrite`
  - `php8.3` (via `libapache2-mod-php` on Ubuntu 24.04)

* * *

## VM Specs (Proxmox)

Recommended Proxmox VM configuration:

| Setting            | Value                            |
| ------------------ | -------------------------------- |
| Name               | `VMINV01`                        |
| Guest OS type      | Linux → Ubuntu 24.x              |
| vCPUs              | 2                                |
| RAM                | 2–4 GB                           |
| Disk               | 20 GB (VirtIO, SSD-backed if possible) |
| NIC model          | VirtIO (paravirtualized)         |
| Firmware           | Default (SeaBIOS or OVMF is fine)|
| QEMU Guest Agent   | **Enabled** in VM → Options      |
| Boot mode          | From Ubuntu Server 24.04 ISO     |

* * *

## 0) Create the VM in Proxmox (no shell needed)

In the Proxmox web UI:

- **Node → Create VM**
  - **General**
    - Node: (choose homelab node)
    - VM ID: (next free)
    - Name: `VMINV01`
  - **OS**
    - Use Ubuntu Server 24.04 ISO.
  - **System**
    - Machine: `q35` (if available)
    - BIOS: default
    - Tick **Qemu Agent** = Enabled.
  - **Disks**
    - Bus/Device: VirtIO SCSI
    - Disk size: `20G` (or slightly larger)
    - Storage: your main VM datastore.
  - **CPU**
    - Cores: `2`
    - Type: `x86-64-v2-AES` (or `host`, consistent with your cluster standard).
  - **Memory**
    - RAM: `4096` MB (can drop to 2048 MB if tight).
  - **Network**
    - Bridge: `vmbr0`
    - Model: `VirtIO (paravirtualized)`

> After creation, boot the VM from the ISO and run through the Ubuntu installer.

* * *

## 1) Ubuntu 24.04 minimal + Guest Agent

During the Ubuntu Server install:

- Select **Ubuntu Server 24.04**.
- Choose **Minimal** install.
- Enable **OpenSSH server** (so we can SSH/SCP later).
- Partition the 20 GB disk as desired (default is fine).
- Set hostname to `vminv01`.
- Create a user (e.g. `ubuntu`) with sudo.

After first boot, SSH into the VM (or use Proxmox console) and do initial prep:

    sudo apt update
    sudo apt -y upgrade

Install and enable the QEMU Guest Agent:

    sudo apt -y install qemu-guest-agent
    sudo systemctl enable --now qemu-guest-agent

> In Proxmox, verify the guest agent is working: VM → Summary → “QEMU Guest Agent” should show as running.

* * *

## 2) Install Apache, PHP, and Tools

Install Apache, PHP, and some basic utilities:

    sudo apt -y install apache2 php libapache2-mod-php curl unzip

Verify Apache is running:

    sudo systemctl status apache2

You should see `active (running)`.

> Ubuntu 24.04 installs PHP 8.3 via `php` and `libapache2-mod-php`. The Apache PHP module is enabled automatically when the package is installed.

* * *

## 3) Upload the Open-AudIT Installer (`.run` file)

The Open-AudIT installer is distributed as a `.run` file, not a `.deb`. We used:

- `OAE-Linux-x86_64-release_5.6.5.run`

### 3.1) Place the installer on your jump host/workstation

Download the `OAE-Linux-x86_64-release_5.6.5.run` file to your local machine or a jump host (follow vendor instructions / portal).

### 3.2) SCP the installer to `vminv01`

From your workstation/jump host (replace `10.0.0.XX` with `vminv01`’s IP if you don’t have DNS):

    scp OAE-Linux-x86_64-release_5.6.5.run ubuntu@vminv01:/home/ubuntu/

Or:

    scp OAE-Linux-x86_64-release_5.6.5.run ubuntu@10.0.0.XX:/home/ubuntu/

On `vminv01`, confirm the file is there:

    cd /home/ubuntu
    ls -lh OAE-Linux-x86_64-release_5.6.5.run

* * *

## 4) Run the Open-AudIT Installer

Make the installer executable and run it with sudo:

    cd /home/ubuntu
    chmod +x OAE-Linux-x86_64-release_5.6.5.run
    sudo ./OAE-Linux-x86_64-release_5.6.5.run

Follow any on-screen prompts. After install, verify the directory and ownership:

    ls -ld /usr/local/open-audit
    ls -l /usr/local/open-audit | head

Expected:

- Installed under `/usr/local/open-audit`.
- Owned by `www-data:www-data` (user and group).

Example:

    drwxr-xr-x 9 www-data www-data 4096 Nov 16 02:40 /usr/local/open-audit

> At this point, Open-AudIT is installed but **not yet wired into Apache**. Browsing to `/open-audit` will not work until we add a site config.

* * *

## 5) Apache Site Config for `/open-audit`

The installer does **not** create an Apache VirtualHost or alias. We need to manually add one.

### 5.1) Enable required Apache modules

Enable URL rewriting and (if needed) the PHP module, then reload Apache:

    sudo a2enmod rewrite
    # PHP module is usually enabled by libapache2-mod-php; if needed:
    # sudo a2enmod php8.3

    sudo systemctl reload apache2

> If you run `a2enmod php8.3` and it errors, it likely just means it’s already enabled or the module name changed — adjust as needed for future Ubuntu releases.

### 5.2) Create the Open-AudIT site config

Create `/etc/apache2/sites-available/open-audit.conf` with the following contents:

    sudo tee /etc/apache2/sites-available/open-audit.conf >/dev/null <<'EOF'
    Alias /open-audit /usr/local/open-audit/public

    <Directory /usr/local/open-audit/public>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
    EOF

This:

- Serves Open-AudIT under `/open-audit`.
- Allows `.htaccess` overrides (needed by the app).
- Grants access to all clients (you can tighten later with firewall / reverse proxy).

### 5.3) Enable the site

Enable the new site and reload Apache:

    sudo a2ensite open-audit
    sudo systemctl reload apache2

(Optional) You can disable the default site if you want Open-AudIT to be the only app:

    # Optional
    sudo a2dissite 000-default
    sudo systemctl reload apache2

* * *

## 6) Validation & First Login

### 6.1) Check Apache config and logs

Sanity check Apache’s configuration:

    sudo apachectl configtest

You should see:

    Syntax OK

If not, fix any errors and re-run the command.

Check for recent Apache errors:

    sudo journalctl -u apache2 --since "10 minutes ago"
    sudo tail -n 50 /var/log/apache2/error.log

### 6.2) Test locally with `curl`

From `vminv01`:

    curl -I http://localhost/open-audit
    curl -I http://localhost/open-audit/

Expected:

- The first may return a `301` or `302` redirect.
- The second should return `200 OK`.

### 6.3) Test from your workstation

In a browser on your workstation, go to:

- `http://<VMINV01_IP>/open-audit/`

For example:

- `http://10.0.0.XX/open-audit/`

You should see the Open-AudIT login page.

### 6.4) First login

Default credentials:

- **Username:** `admin`
- **Password:** `password`

> Immediately change the default password under the Open-AudIT UI settings once you can log in.

From here you can:

- Configure device discovery.
- Add scan credentials.
- Integrate with the rest of your homelab inventory.

* * *

## 7) Troubleshooting (Quick Hits)

### 7.1) 403 Forbidden at `/open-audit` or `/open-audit/`

**Symptoms:**

- Browser shows `403 Forbidden`.
- `curl -I http://localhost/open-audit/` returns `403`.

**Checklist:**

1. **Site file exists and is enabled**

    - Confirm config file:

          ls -l /etc/apache2/sites-available/open-audit.conf

    - Confirm `Alias` and `<Directory>` block are present and match:

          Alias /open-audit /usr/local/open-audit/public

          <Directory /usr/local/open-audit/public>
              Options Indexes FollowSymLinks
              AllowOverride All
              Require all granted
          </Directory>

    - Confirm site is enabled:

          ls -l /etc/apache2/sites-enabled | grep open-audit

      or:

          sudo a2ensite open-audit
          sudo systemctl reload apache2

2. **Permissions on `/usr/local/open-audit`**

    - Check owner and mode:

          ls -ld /usr/local/open-audit
          ls -ld /usr/local/open-audit/public

      Expect `www-data:www-data` and at least `755` on directories.

3. **Apache logs**

    - Check for permission or directory-related errors:

          sudo tail -n 50 /var/log/apache2/error.log

    Fix any obvious `AHxxxx` errors (permissions, path typos) and reload Apache.

### 7.2) Blank page, 500 errors, or PHP issues

**Symptoms:**

- 500 Internal Server Error.
- Blank page when hitting `/open-audit/`.
- PHP-related errors in logs.

**Checklist:**

1. **Confirm PHP and Apache PHP module are installed**

    - Confirm packages:

          dpkg -l | grep -E 'php|libapache2-mod-php'

    - If missing, install:

          sudo apt -y install php libapache2-mod-php
          sudo systemctl restart apache2

2. **Ensure PHP module is enabled**

    - Check modules:

          a2query -m | grep php

    - If not present or disabled, enable it (Ubuntu 24.04 example):

          sudo a2enmod php8.3
          sudo systemctl restart apache2

3. **Check Apache error log for PHP stack traces**

    - Look for fatal errors:

          sudo tail -n 100 /var/log/apache2/error.log

    Resolve any missing extensions or misconfigurations as indicated.

### 7.3) 404 Not Found at `/open-audit`

**Symptoms:**

- Browser shows `404 Not Found`.
- `curl -I http://localhost/open-audit/` returns 404.

**Checklist:**

1. **Alias path correct**

    - Confirm the path in the site config matches actual install location:

          ls -ld /usr/local/open-audit/public

    - If the installer used a different path (unusual), update the `Alias` and `<Directory>` paths to match.

2. **Site enabled**

    - Ensure the `open-audit` site is enabled:

          sudo a2ensite open-audit
          sudo systemctl reload apache2

3. **No conflicting configs**

    - Check for conflicting `Alias /open-audit` definitions in other site files under `/etc/apache2/sites-available` or `/etc/apache2/conf-available`.

### 7.4) Port / Firewall Issues

If you can curl `http://localhost/open-audit/` on `vminv01` but not reach it from your workstation:

1. **Apache listening on port 80**

    - Confirm:

          sudo ss -tlnp | grep ':80'

2. **UFW or other firewall**

    - If UFW is enabled, allow HTTP:

          sudo ufw status
          sudo ufw allow 80/tcp

3. **Proxmox bridge / VLAN**

    - Ensure `vmbr0` and any VLANs are correctly configured so your workstation can reach `vminv01`’s IP.

* * *

## 8) Post-Install Notes

- **Backups:**  
  Add `VMINV01` to your Proxmox backup job(s) (vzdump) and keep configs under version control (e.g., store `/etc/apache2/sites-available/open-audit.conf` in Git).
  
- **Monitoring:**  
  Consider adding `vminv01` to your monitoring stack (`vmmon01` → Prometheus/node_exporter) so you can track CPU/RAM/disk usage.

- **Integration:**  
  Forward Open-AudIT logs or export inventory data to `vmgray01` (Graylog) or your other tooling as needed.

This completes the Open-AudIT Community deployment for `VMINV01`.
