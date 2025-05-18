---
title: Building a WireGuard Site-to-Site Tunnel
date: 2025-05-18 12:45:00 -0400
categories: [Networking, VPN, WireGuard]
tags: [WireGuard, Raspberry Pi, Linode, VPS, Site-to-Site]
---
## Why I Did This

I wanted to setup WireGuard so I can connect to my home network when on the road. Instead of directly connecting to my home network while on public or unsecure connections, I set up a "jumpbox" with Linode. 

I also wanted to avoid Dynamic DNS or port forwards on my home router. I setup a DNS record with Cloudflare to my VPS' public static IPv4 address. 

Also using a Raspberry Pi 5. The Pi acts as a WireGuard client and maintains a persistent tunnel to the VPS, which acts as the bridge to my home LAN. This guide will show you how to set up a secure tunnel quickly.

---

## What You’ll Need

* Linode VPS (or a VPS of your choosing, ideally in a region close to you)
* Raspberry Pi 5 running latest Raspberry Pi OS Lite
* About 30 minutes
* Optional: DNS record of your VPS static IP for WireGuard's "Endpoint" setting.

---

## Step 1: Setup and Harden the VPS

After you have your VPS running, we'll need to SSH in and get it set up before touching WireGuard. You can use a Linux terminal or PuTTY on Windows.

### Update and create non-root user:

```bash
sudo apt update && sudo apt upgrade -y
sudo adduser mark
sudo usermod -aG sudo mark
```
We'll be using the regular user "mark" for everything. The user will need admin privileges, so we've added it to the sudo group.

### SSH hardening:

```bash
# Create your own SSH keys, then add the public key to ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

Edit `/etc/ssh/sshd_config` and look for these three options. Ensure that they are not commented out, and that the last two have the value of "no" before saving, like so:

```bash
Port 65535  # Changed from default of 22
PermitRootLogin no
PasswordAuthentication no
X11Forwarding no  # Unnecessary to have it enabled since VPS is headless
```

> SSH uses the default port of 22 which is frequently targeted by scanners and attackers. It is recommended that you pick a random high port instead!
>
> The option "PasswordAuthentication no" will require public key authentication. Make sure you add your pubkey to the authorized_keys file, and that "PubkeyAuthentication yes" is in your sshd_config file!
{: .prompt-warning }

Reload SSH:

```bash
sudo systemctl reload ssh
```

### Lock root user's password:

```bash
sudo passwd -l root
```
This will prevent the root user from logging in. Since we've added the "mark" user to the sudo group, they will still be able to execute admin commands as if root. This step is optional.

### Set timezone:

```bash
sudo timedatectl set-timezone America/New_York
sudo timedatectl set-ntp true
```
> Adjust to your timezone as needed. You can use ```timedatectl list-timezones``` to find yours.
{: .prompt-tip }

### Enable unattended upgrades:

```bash
sudo apt install unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
sudo nano /etc/apt/apt.conf.d/50unattended-upgrades
```

Make sure these lines were not commented out with the leading "//".

```bash
"origin=Debian,codename=${distro_codename}-updates";
"origin=Debian,codename=${distro_codename},label=Debian";
"origin=Debian,codename=${distro_codename},label=Debian-Security";
"origin=Debian,codename=${distro_codename}-security,label=Debian-Security";
```

These two lines will cause the VPS to automatically reboot at 4:30 AM if a reboot is required after upgrading.

```bash
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-Time "4:30";
```

### UFW firewall:

```bash
sudo apt install ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 65535/tcp comment 'SSH'    # Your custom SSH port
sudo ufw allow 51820/udp comment 'WireGuard'    # Default WireGuard port
sudo ufw enable
```
> WireGuard uses the default port of 51820. Again recommending you pick a random high port instead.
{: .prompt-warning }

You can optionally use Linode’s cloud firewall as well to only allow these same ports. Block everything else. I decided not to use Linode's firewall since WireGuard will be adding to iptables during setup and teardown of the tunnel.

---

## Step 2: Check for Unnecessary Open Ports

Use `ss` to check for listening services:

```bash
sudo ss -tulnp
```

Look for any ports that are open and listening, not bound to localhost. Here are some examples:

* `:5355` → systemd-resolved / Link-Local Multicast Name Resolution (closed this port)
* `:25` → mail services (close it unless you're running a mail server)
* `:323` → chronyd (safe if bound to localhost only)

On Linode VPS, only port 323 was open and bound to localhost, so I left it alone. 
Close or firewall anything you’re not explicitly using. 

---

## Step 3: Install and Configure WireGuard

### On Linode VPS:

```bash
sudo apt install wireguard
sudo -i     # This will switch you to root so you can access WireGuard's directory easily
cd /etc/wireguard
umask 077
wg genkey | tee server_private.key | wg pubkey > server_public.key
```

Create `/etc/wireguard/wg0.conf` with Nano or Vim:

```ini
[Interface]
Address = 10.10.0.1/29
ListenPort = 51820
PrivateKey = <Linode private key>
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT
PostUp = iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT
PostDown = iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = <Pi public key>
AllowedIPs = 10.10.0.2/32, 192.168.100.0/24
PersistentKeepalive = 25
```
> Ensure you use the correct listening port you set up in UFW. You may have to change the interface of "eth0" if you are using a different OS. Make sure you put your correct private network address and netmask in the AllowedIPs of the Peer section.
{: .prompt-warning }

In this configuration, I chose the IP 10.10.0.1 for the VPS on the WG tunnel with a netmask of /29. Since I am not using that many devices at once on the network, this will lock down the total usable hosts to 6 (excluding network address and broadcast). You can change that if you want more devices to be able to connect to WireGuard.

Make sure you add the actual contents of the private key into the PrivateKey section, not the file path. You'll have to add the Pi's public key that is created later under the Peer section of this configuration. We're assigning it the IP of 10.10.0.2 while adding the local network address I have setup at home of 192.168.100.0/24.

---

### On the Raspberry Pi:

```bash
sudo apt install wireguard
```

Re-use existing keys, or generate new ones using same steps as above.

Create `/etc/wireguard/wg0.conf`:

```ini
[Interface]
Address = 10.10.0.2/29
ListenPort = 51820
PrivateKey = <Pi private key>
PostUp   = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = <Linode public key>
Endpoint = <linode_public_ip>:51820
AllowedIPs = 10.10.0.0/29   # Allows the Pi to communicate with other devices on the WG subnet without tunneling all outbound traffic through the VPS
PersistentKeepalive = 25
```

---

## Step 4: Start and Enable WireGuard

On both Linode and the Pi:

```bash
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
```

Check status and handshakes:

```bash
sudo wg show
```

Ping across the tunnel:

```bash
ping 10.10.0.1   # from Pi
ping 10.10.0.2   # from VPS
```

---

## Step 5: Optimize MTU for Mobile Devices

If using WireGuard on mobile device (e.g., Verizon/AT&T on 5G), set:

```ini
MTU = 1320
```

This should help prevent packet fragmentation and random slowdowns over LTE/5G. Keep the default MTU (1420) on Pi and VPS unless you're troubleshooting. The MTU of 1320 should only be set on your mobile device with WireGuard configured.

---

## Step 6: Real-World Performance

When I switched from DigitalOcean (New York) to Linode (Atlanta), I went from \~80 Mbps to 160+ Mbps over Verizon 5G UW, with ping dropping to \~36 ms and jitter nearly gone. Performance varies by cell signal strength (as expected), but the tunnel itself is solid.

If I weren’t routing through Linode as a jumpbox, I’d likely get even higher speeds. Ultimately decided that this was a fair tradeoff for privacy. Even though the Pi 5 did not show a CPU bottleneck, I'm sure a dedicated network appliance designed for high VPN throughput would show significantly better speeds than the Pi. Maybe I'll take advantage of my 1 Gb internet speeds by incorporating a pfSense or Protectli Vault box in the future.

---

## Mobile Device WireGuard Config Example

```ini
[Interface]
PrivateKey = <your_phone_private_key>
Address = 10.10.0.3/29
DNS = 192.168.100.1
MTU = 1320

[Peer]
PublicKey = <linode_server_public_key>
Endpoint = <linode_public_ip>:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

---

## Final Thoughts

Want even more flexibility? Add a second VPS in another different region, and use multiple WireGuard configs on your phone or laptop to switch based on your current location. I may cover multi-region/failover configs in a future post.
