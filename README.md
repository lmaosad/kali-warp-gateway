#  Turn Any Old PC or Laptop into an Encrypted Hardware WARP Gateway
## Installtion files for devices will be included in the files above if you want to try out cloudflare one or cloudflare warp.
---
Give your old hardware a powerful second life! This project demonstrates how to upcycle **any old PC, laptop, mini-PC, or Raspberry Pi** into a dedicated, hardware-level **Security & VPN Gateway** using **Kali Linux** and **Cloudflare WARP**.

By placing this gateway machine between your main work/gaming PC (or entire home subnet) and the internet, all outbound network traffic gets transparently routed, encrypted, and protected via Cloudflare WARP with zero performance-robbing routing loops or packet drops.

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
[ Main PC 1 ] ----\
[ Main PC 2 ] ----->  [ (Optional) Old Router as Switch ] ---> (Ethernet) ---> [ Gateway (Kali PC) ] ---> (Wi-Fi + WARP) ---> [ Internet ]
[ Smart TV  ] ----/         (DHCP Disabled)                                        10.42.0.1
```

---

##  What to Install on the Gateway Machine

> [!IMPORTANT]
> **Cloudflare Repository Setup Notice:** Kali Linux (and Debian-based OSes) do not include Cloudflare WARP in standard default repositories. You **must add the Cloudflare GPG key and APT repository list** before running `apt install cloudflare-warp`, otherwise APT will fail to locate the package.

Open a terminal on your Kali Linux gateway machine and run the following commands:

```bash
# 1. Update system packages and install prerequisites
sudo apt update && sudo apt install curl gnupg lsb-release iptables net-tools -y

# 2. Add Cloudflare GPG key to trusted keyrings
curl -fsSL [https://pkg.cloudflareclient.com/pubkey.gpg](https://pkg.cloudflareclient.com/pubkey.gpg) | sudo gpg --yes --dearmor --output /usr/share/keyrings/cloudflare-warp-archive-keyring.gpg

# 3. Add the Cloudflare repository to APT package sources list
echo "deb [signed-by=/usr/share/keyrings/cloudflare-warp-archive-keyring.gpg] [https://pkg.cloudflareclient.com/](https://pkg.cloudflareclient.com/) $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/cloudflare-client.list

# 4. Refresh package lists and install Cloudflare WARP
sudo apt update
sudo apt install cloudflare-warp -y

# 5. Register Cloudflare WARP Client
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

##  Bonus: Connect Multiple Devices using an Old Spare Router as a Switch

If you have an extra Wi-Fi router lying around unused, you can turn it into an unmanaged network switch / secondary access point to route **multiple devices** (PCs, consoles, TVs, phones) through your Kali WARP Gateway!

### How to configure the spare router:
1. **Log into the Spare Router's Admin Panel:** Connect to the old router and access its settings (usually `192.168.1.1` or `192.168.0.1`).
2. **Disable the DHCP Server:** Turn off the DHCP Server in the router settings. This prevents the spare router from handing out conflicting IP addresses.
3. **Change Router IP (Optional):** Set the router's local management IP to `10.42.0.2` so it sits safely inside your Kali gateway's subnet.
4. **Physical Cabling:**
   * Plug an Ethernet cable from the **Kali Gateway Ethernet Port** into **LAN Port 1** on the spare router (*Do NOT use the WAN/Internet port*).
   * Plug your Main Desktop, Gaming Console, or other PCs into **LAN Ports 2, 3, and 4**.
   * *(Optional)* Leave Wi-Fi enabled on the spare router if you want encrypted WARP Wi-Fi for mobile devices!

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

Save this script on your Kali machine to apply all rules and repo additions in one command:

```bash
#!/bin/bash
ETH_IF="eth0"
WIFI_IF="wlan1" # Change to your active WAN interface

echo "[+] Checking Cloudflare WARP Repository..."
if ! command -v warp-cli &> /dev/null; then
    echo "[+] Adding Cloudflare GPG key and APT repository..."
    sudo apt update && sudo apt install curl gnupg lsb-release -y
    curl -fsSL [https://pkg.cloudflareclient.com/pubkey.gpg](https://pkg.cloudflareclient.com/pubkey.gpg) | sudo gpg --yes --dearmor --output /usr/share/keyrings/cloudflare-warp-archive-keyring.gpg
    echo "deb [signed-by=/usr/share/keyrings/cloudflare-warp-archive-keyring.gpg] [https://pkg.cloudflareclient.com/](https://pkg.cloudflareclient.com/) $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/cloudflare-client.list
    sudo apt update && sudo apt install cloudflare-warp -y
    warp-cli registration new
fi

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

echo "[+] Connecting to Cloudflare WARP..."
warp-cli connect

echo "[+] Hardware Gateway rules active & WARP connected!"
```
