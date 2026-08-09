# 🛠️ Turn Any Old PC or Laptop into an Encrypted Hardware WARP Gateway

Give your old hardware a powerful second life! This project demonstrates how to upcycle **any old PC, laptop, mini-PC, or Raspberry Pi** into a dedicated, hardware-level **Security & VPN Gateway** using **Kali Linux** and **Cloudflare WARP**.

By placing this gateway machine between your main work/gaming PC and the internet, all outbound network traffic gets transparently routed, encrypted, and protected via Cloudflare WARP with zero performance-robbing routing loops or packet drops.

---

##  How I Came Up With This Idea

I wanted a way to run Cloudflare WARP encryption for my main desktop without having to keep virtual network adapters, VPN background daemons, or strict firewall apps clogging up my main OS. 

I realized I had extra hardware lying around. Instead of buying an expensive physical router, I wired an Ethernet cable directly from my main PC to an old machine running Kali Linux. By turning that secondary machine into a dedicated router/NAT bridge, my main PC gets full encrypted internet access with zero local CPU overhead or software interference!

---

##  Why Kali Linux for This Specific Setup?

While any Linux distribution (like Ubuntu or Debian) can technically forward packets, **Kali Linux is uniquely suited** for this setup out-of-the-box:

1. **Built-in Networking & Routing Tools:** Kali comes pre-loaded with full `iptables`, `nftables`, `net-tools`, and advanced driver suites right out of the box.
2. **Easy Headless & Hotspot Management:** Kali's `NetworkManager` stack makes bridged connection setup (`10.42.0.x`) automatic as soon as an Ethernet cable is plugged in.
3. **Security Testing Ready:** Having a dedicated Kali gateway allows you to easily inspect, monitor, or capture network traffic (`Wireshark`, `tcpdump`) passing through your main PC at the hardware layer before it hits the internet.

---

##  Network Architecture

```text
[ Main Desktop PC ]  --->  (Direct Ethernet Cable)  --->  [ Gateway (Old PC/Laptop) ]  --->  (Wi-Fi / wlan1 + WARP)  --->  [ Internet ]
   10.42.0.28                                                    10.42.0.1
```

---

##  What to Install on the Gateway Machine

On your Kali Linux machine, open a terminal and run:

```bash
# 1. Update package lists
sudo apt update

# 2. Install iptables and network utilities
sudo apt install iptables net-tools curl -y

# 3. Install Cloudflare WARP Client
sudo apt install cloudflare-warp -y

# 4. Register Cloudflare WARP
warp-cli registration new
```

---

##  Step-by-Step Commands & Technical Breakdown

When connecting a client to WARP over a local bridge, standard masquerading creates **routing loops** and **packet drops** (often disconnecting after 10 seconds). 

Here is every exact command executed to fix the setup and why it was used:

### 1. Enable Kernel Packet Forwarding
Allows the Linux kernel to act as a router and forward incoming packets from Ethernet out to Wi-Fi.
```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

### 2. Disable Reverse Path Filtering (`rp_filter`)
*Crucial fix!* Cloudflare WARP modifies interface routing rules. If Reverse Path Filtering is enabled, the kernel thinks forwarded packets are spoofed and drops them immediately.
```bash
sudo sysctl -w net.ipv4.conf.all.rp_filter=0
sudo sysctl -w net.ipv4.conf.eth0.rp_filter=0
sudo sysctl -w net.ipv4.conf.wlan1.rp_filter=0
```

### 3. Clear Restrictive Firewall Rules
Clears existing firewall restrictions to prevent policy blocks:
```bash
sudo iptables -F FORWARD
sudo iptables -t nat -F
```

### 4. Clamp TCP MSS (Fixes Packet Fragmentation & Slow Speeds)
WARP adds encryption headers to every packet, causing standard 1500-byte packets to fragment and chop download speeds. Clamping the TCP MSS size to `1360` bytes ensures maximum network throughput without packet loss.
```bash
sudo iptables -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --set-mss 1360
```

### 5. Configure Strict Interface NAT (Prevents 10-Second Crashes)
Restricts NAT masquerading *strictly* to the Wi-Fi adapter (`wlan1`). This prevents WARP from looping packets back into Ethernet and crashing the tunnel.
```bash
sudo iptables -A FORWARD -i eth0 -o wlan1 -j ACCEPT
sudo iptables -A FORWARD -i wlan1 -o eth0 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -t nat -A POSTROUTING -o wlan1 -j MASQUERADE
```

### 6. Connect WARP
```bash
warp-cli connect
```

---

##  Main PC Configuration (Client)

1. Connect an Ethernet cable between your main PC and the Gateway machine.
2. Set your Main PC Ethernet IPv4 properties:
   * **IP Address:** `10.42.0.28` (or set adapter to DHCP)
   * **Subnet Mask:** `255.255.255.0`
   * **Default Gateway:** `10.42.0.1` (Kali's Ethernet IP)
   * **Preferred DNS:** `1.1.1.1`
   * **Alternate DNS:** `8.8.8.8`

---

##  How People Can Improve This Setup

If you want to take this project further, here are the best upgrades:

1. **Use a USB 3.0 to Ethernet Adapter (Dual Ethernet Gateway):**
   * Instead of receiving internet over Wi-Fi, plug a second Ethernet adapter into the old PC. Passing traffic `Ethernet -> Ethernet` eliminates wireless latency and unlocks Gigabit wire speeds.
2. **Upgrade to a 5GHz / Wi-Fi 6 USB Dongle:**
   * Built-in Wi-Fi chips on older laptops/PCs are often capped at 2.4GHz (~10-15 Mbps). Plugging in a $10–$15 5GHz USB Wi-Fi adapter (`wlan1`) immediately boosts speeds past 100+ Mbps.
3. **Automate on Boot (`systemd` / `cron`):**
   * Add the iptables script to system startup (`/etc/rc.local` or a `systemd` service) so your hardware VPN powers up automatically whenever you turn on the old machine.
4. **Use Any Old Hardware:**
   * This setup isn't limited to laptops! Old desktop towers, Intel NUCs, mini PCs, or Raspberry Pis make incredible silent hardware gateways.

---

##  Automated Setup Script (`setup-gateway.sh`)

Save this script on your Kali machine to apply all rules in one command:

```bash
#!/bin/bash
ETH_IF="eth0"
WIFI_IF="wlan1" # Change to your active WAN interface

echo "[+] Enabling Kernel Packet Forwarding..."
sudo sysctl -w net.ipv4.ip_forward=1 > /dev/null

echo "[+] Disabling Reverse Path Filtering..."
sudo sysctl -w net.ipv4.conf.all.rp_filter=0 > /dev/null
sudo sysctl -w net.ipv4.conf.${ETH_IF}.rp_filter=0 > /dev/null
sudo sysctl -w net.ipv4.conf.${WIFI_IF}.rp_filter=0 > /dev/null

echo "[+] Setting up NAT Rules & TCP MSS Clamping..."
sudo iptables -F FORWARD
sudo iptables -t nat -F
sudo iptables -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --set-mss 1360
sudo iptables -A FORWARD -i ${ETH_IF} -o ${WIFI_IF} -j ACCEPT
sudo iptables -A FORWARD -i ${WIFI_IF} -o ${ETH_IF} -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -t nat -A POSTROUTING -o ${WIFI_IF} -j MASQUERADE

echo "[+] Hardware Gateway rules active! Connect WARP with 'warp-cli connect'."
```
