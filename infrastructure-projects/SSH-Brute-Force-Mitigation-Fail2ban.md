# Implementing Fail2ban Intrusion Prevention on a Linux server with UFW & OpenVPN

This guide documents the implementation of **Fail2ban** on a Linux server. The server is already secured with **UFW (Uncomplicated Firewall)**, **OpenVPN**, and a **network-layer killswitch** to prevent IP leaks. 

Adding Fail2ban introduces a *dynamic* layer of security. While UFW keeps your ports managed, Fail2ban actively scans logs to dynamically ban IPs that attempt brute-force attacks against open local management interfaces (like SSH or application WebUIs).

---

## Architecture Overview

```
[ Attacker / Brute-Force Bot ] 
              │
              ▼
    [ UFW Open Ports ] (SSH, WebUIs, Local Subnet)
              │
              ▼
   [ Log Files Edited ] (e.g., /var/log/auth.log)
              │
              ▼
   [ Fail2ban Daemon ] ──(Detects Max Retries)──► [ Automates UFW Deny Rule ] ──► [ IP Blocked 🚫 ]
```

---

## Step 1: Install Fail2ban

Since UFW is already enabled and managing the VPN killswitch, installing Fail2ban is straightforward.

```bash
sudo apt update
sudo apt install fail2ban -y
```
![Fail2ban Install](https://github.com/k3rn3l-p4n1k-l4b/10inch-rack-homelab-infrastructure/blob/main/images/Fail2Ban%20Download.jpg)

Verify that the service is running successfully:
```bash
sudo systemctl status fail2ban
```
![Fail2Ban Systemctl Status](https://github.com/k3rn3l-p4n1k-l4b/10inch-rack-homelab-infrastructure/blob/main/images/Fail2Ban%20systemctl%20Status.jpg)

---

## Step 2: Configure the Local Jail

Never edit the default `/etc/fail2ban/jail.conf` file, as updates will overwrite it. Instead, create a `.local` copy:

```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

Open the configuration file in your editor:
```bash
sudo nano /etc/fail2ban/jail.local
```

### Critical Settings for a Headless Linux Server
Scroll down to the `[DEFAULT]` section. Modify or add the following lines. 

*Note: It is crucial to whitelist your home network's local subnet in `ignoreip` so you don't accidentally lock yourself out of your headless box.*

```ini
[DEFAULT]
# Whitelist local loopback and your home network subnet (adjust to match your LAN IP scheme)
ignoreip = 127.0.0.1/8 ::1 XXX.XXX.XX.0/24 XX.X.X.X/24 xxx.xxx.xx.x

# Ban settings
bantime  = 1h
findtime = 10m
maxretry = 5

# Tell Fail2ban to use UFW as the backend firewall instead of iptables
banaction = ufw
banaction_allports = ufw
```

Scroll down further to enable the SSH daemon protection jail:

```ini
[sshd]
enabled = true
banaction =  ufw
#mode=normal
port    = ssh
logpath = %(sshd_log)s
backend = %(sshd_backend)s
```

Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`).

---

## Step 3: Apply Changes & Verify Integration

Restart Fail2ban to load your new `jail.local` rules:

```bash
sudo systemctl restart fail2ban
```

Now, check the server-wide status of Fail2ban to confirm your SSH jail is actively monitoring:

```bash
sudo fail2ban-client status
```
![Fail2Ban Client Status](https://github.com/k3rn3l-p4n1k-l4b/10inch-rack-homelab-infrastructure/blob/main/images/fail2ban%20status.jpg)

---

## Step 4: Testing the Setup (Simulating an Attack)

To safely test this on your headless machine without locking yourself out, you can simulate an attack from a separate machine on your local network—**provided that machine's IP is NOT in your `ignoreip` whitelist**.

1. From a secondary client machine, attempt to SSH into your Linux server using an invalid username or password multiple times:
   ```bash
   ssh fakeuser@<your_linux_server_ip>
   ```
2. Repeat this until the connection is completely refused.

### Checking the Ban Status
Back on your headless Linux server, check the specific metrics of the SSH jail:

```bash
sudo fail2ban-client status sshd
```
![Fail2Ban-client status sshd](https://github.com/k3rn3l-p4n1k-l4b/10inch-rack-homelab-infrastructure/blob/main/images/f2b%20status%20sshd.jpg)

### Verifying UFW Dynamic Injection
Because we specified `banaction = ufw`, look at your firewall rules to see Fail2ban's automated rule injection:

```bash
sudo ufw status numbered
```

Fail2ban will insert a raw `DENY` rule at the very top of your UFW chain, overriding any other rules until the ban timer expires.

![Fail2Ban Ban Status](https://github.com/k3rn3l-p4n1k-l4b/10inch-rack-homelab-infrastructure/blob/main/images/Fail2Ban%20UFW%20RULE.jpg)

## Emergency Flush (Unban Everything)

If multiple devices are locked out simultaneously during testing or maintenance, you can clear all active bans across all operational jails at once:

```bash
sudo fail2ban-client unban --all
```
---

## Conclusion
Your headless Linux server now features dynamic perimeter security. While UFW protects the VPN interface from leaking traffic outwards, Fail2ban prevents inbound intruders from taking advantage of open local management utilities.
