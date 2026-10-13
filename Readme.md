# Comprehensive Guide: Tor Proxy Setup with 30-Second IP Rotation on Debian

Welcome! This guide provides a full, step-by-step walkthrough for absolute beginners on how to configure **Tor** on **Debian Linux** to route traffic through an anonymizing proxy, set up a simple toggle switch (`torswitch`), and configure automatic or manual **30-second IP rotation** using Python and Stem.

---

## Table of Contents
1. [Prerequisites & Understanding the Setup](#1-prerequisites--understanding-the-setup)
2. [Step 1: Installing and Configuring Tor (`torrc`)](#2-step-1-installing-and-configuring-tor-torrc)
3. [Step 2: Starting and Verifying Tor Services](#3-step-2-starting-and-verifying-tor-services)
4. [Step 3: Testing the SOCKS5 Proxy](#4-step-3-testing-the-socks5-proxy)
5. [Step 4: Implementing the `torswitch` Control Script](#5-step-4-implementing-the-torswitch-control-script)
6. [Step 5: Setting Up 30-Second IP Rotation with Python & Stem](#6-step-5-setting-up-30-second-ip-rotation-with-python--stem)
7. [Step 6: Automated Loop Script for Continuous Rotation](#7-step-6-automated-loop-script-for-continuous-rotation)
8. [GitHub Upload Instructions](#8-github-upload-instructions)

---

## 1. Prerequisites & Understanding the Setup

Before we begin, let's understand what ports Tor uses:
* **Port `9050`**: Standard SOCKS5 proxy port used by applications/clients to route traffic through Tor.
* **Port `9040`**: Transparent proxy port (used for system-wide routing via iptables/nftables if enabled via `torswitch`).
* **Port `9051`**: Control port used to send administrative commands to Tor (such as requesting a new identity/IP via `NEWNYM`).
* **Port `5354`**: DNS port for resolving domain names securely through Tor.

---

## 2. Step 1: Installing and Configuring Tor (`torrc`)

First, ensure Tor is installed on your Debian system:
```bash
sudo apt update
sudo apt install tor python3-stem curl -y
```

Next, configure your `/etc/tor/torrc` file to expose the SOCKS port, DNS port, and Control port with cookie authentication enabled:
```bash
sudo nano /etc/tor/torrc
```

Make sure these lines are present and uncommented:
```text
SocksPort 9040
SocksPort 9050
DNSPort 5354
ControlPort 9051
CookieAuthentication 1
```

Verify your configuration syntax is valid:
```bash
sudo tor --verify-config -f /etc/tor/torrc
```

---

## 3. Step 2: Starting and Verifying Tor Services

Restart and enable the Tor systemd service:
```bash
sudo systemctl restart tor@default
sudo systemctl enable tor@default
```

Check if the ports are actively listening:
```bash
sudo ss -lntup | grep -E ':(9050|9051|9040|5354)\b'
```
You should see processes owned by `debian-tor` listening on these ports.

---

## 4. Step 3: Testing the SOCKS5 Proxy

To verify that your traffic is successfully passing through Tor, run a curl request through the SOCKS5 proxy:
```bash
curl --proxy socks5h://127.0.0.1:9050 --max-time 60 https://check.torproject.org/api/ip
```
**Expected Output:**
```json
{"IsTor":true,"IP":"<some-tor-exit-ip>"}
```

---

## 5. Step 4: Implementing the `torswitch` Control Script

To easily enable and disable system-wide or proxy configurations, create a custom helper script named `torswitch`:

```bash
sudo nano /usr/local/bin/torswitch
```

Paste the following script content:

```bash
#!/bin/bash
case "$1" in
    on)
        echo "Starting Tor..."
        sudo systemctl start tor@default
        echo "Tor system-wide routing is ON."
        echo "IPv4 TCP is routed through Tor; IPv4 UDP DNS uses Tor DNSPort."
        ;;
    off)
        sudo systemctl stop tor@default
        echo "Tor daemon stopped. No system-wide rules active."
        ;;
    status)
        if systemctl is-active --quiet tor@default; then
            echo "active"
            sudo ss -lntup | grep -E ':(9050|9051|9040)\b'
            curl -sS --max-time 10 https://check.torproject.org/api/ip
            echo -e "\nTor daemon and SOCKS proxy are ready."
        else
            echo "inactive"
        fi
        ;;
    *)
        echo "Usage: torswitch {on|off|status}"
        exit 1
        ;;
es:
```

Make the script executable:
```bash
sudo chmod +x /usr/local/bin/torswitch
```

Now you can control Tor easily using:
```bash
torswitch status
torswitch on
torswitch off
```

---

## 6. Step 5: Setting Up 30-Second IP Rotation with Python & Stem

To change your public exit IP address programmatically without restarting the Tor daemon, we use Python with the `stem` library interacting with the ControlPort (`9051`).

Run the following one-liner command to request a new circuit (`NEWNYM` signal):
```bash
sudo -u debian-tor /usr/bin/python3 - <<'PY'
from stem import Signal
from stem.control import Controller
with Controller.from_port(address="127.0.0.1", port=9051) as controller:
    controller.authenticate()
    controller.signal(Signal.NEWNYM)
    print("Tor accepted the NEWNYM request.")
PY
```

---

## 7. Step 6: Automated Loop Script for Continuous Rotation

To rotate your IP address automatically every 30 seconds while checking your current exit IP, use this continuous bash/python loop:

```bash
while true; do
    echo "===== $(date '+%H:%M:%S') ====="
    sudo -u debian-tor /usr/bin/python3 -c '
from stem import Signal
from stem.control import Controller
with Controller.from_port(address="127.0.0.1", port=9051) as c:
    c.authenticate()
    c.signal(Signal.NEWNYM)
    print("New circuit requested")
'
    curl -sS --max-time 25 https://check.torproject.org/api/ip
    echo
    sleep 30
done
```

---

## 8. GitHub Upload Instructions

To upload this guide and your scripts to GitHub, follow these commands in your project folder:

1. **Initialize Git:**
   ```bash
   git init
   ```

2. **Add this README file:**
   ```bash
   git add README.md
   ```

3. **Commit your changes:**
   ```bash
   git commit -m "Add comprehensive Tor proxy and 30s IP rotation guide for Debian"
   ```

4. **Link to your GitHub Repository and Push:**
   ```bash
   git branch -M main
   git remote add origin https://github.com/YOUR_USERNAME/YOUR_REPOSITORY_NAME.git
   git push -u origin main
   ```

*(Replace `YOUR_USERNAME` and `YOUR_REPOSITORY_NAME` with your actual GitHub details).*
