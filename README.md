# OpenWrt Configuration for Fast Roaming

Configuration guide and reference settings for Belkin RT3200 / Linksys E8450 routers running OpenWrt 25.12.

The configuration is built around two core ideas:

* **Fast Roaming (802.11k/v + DAWN, without 802.11r):** Client handoffs are managed using 802.11k (Neighbor Reports) and 802.11v (BSS Transition Management) and then coordinated across APs by DAWN for intelligent band and AP steering. We omit 802.11r (Fast BSS Transition) because it introduces mobility-domain synchronization issues, all for a possible savings of 10–20 ms. And modern Apple, Android and Windows devices can roam seamlessly with 802.11k/v alone.
* **modern security & device isolation (WPA3-SAE Only + IoT VAP):** WPA3-Personal (`sae` only) eliminates the security vulnerabilities and negotiation bugs common to mixed WPA2/WPA3 (`sae-psk`) networks. We don't need backward compatibility because the Wi-Fi Alliance mandated WPA3 on devices certified after July 1, 2020. For smart home and legacy equipment that require WPA2-PSK and standard 2.4 GHz rates, we provide a dedicated virtual AP (VAP) with client isolation.

---

## Architecture: Gateway vs. Access Point

This deployment separates router responsibilities into two distinct roles:

* **Gateway Router (1 device):**
  * Handles NAT routing, firewall rules, DHCP/DNS services (`dnsmasq` for IPv4 and `odhcpd` for IPv6).
  * Connected directly to the internet port (WAN).
* **Access Points (1 or more devices):**
  * Configured as bridged "dumb APs".
  * Routing, firewall and DHCP/DNS services are disabled.
  * All ethernet ports and wireless interfaces are bridged directly to the gateway's LAN network.

### Belkin RT3200 Hardware Highlights

This configuration relies on several distinctive hardware features of the Belkin RT3200.

* **Linux DSA (Distributed Switch Architecture):** The built-in MT7531 Gigabit switch is driven by the modern Linux DSA subsystem. Unlike older routers that rely on `swconfig` and complex VLAN tagging, each physical port (`wan`, `lan1`–`lan4`) is exposed as an individual network interface.
* **`mt76` driver:** The MT7915 4x4 Wi-Fi 6 radio is driven by the open source `mt76` kernel driver. This provides first-class `mac80211` integration, reliable 802.11k/v signaling and accurate DAWN RSSI telemetry.
* **DFS radar sensitivity:** The MT7915 radio has sensitive radar detection circuitry, causing frequent false radar triggers on DFS bands. This hardware behavior motivates the non-DFS channel plan described later in [5G Channel Plan](#5g-channel-plan).

### OpenWrt 25.12 Features

This setup incorporates modern OpenWrt 25.x conventions and optimization practices that differ from stock defaults or older OpenWrt releases:

* **Explicit Regulatory Domain (`country 'US'`):** Default installations leave this unset (defaulting to the restrictive world domain `00`). Explicitly configuring the country unlocks full legal transmit power (EIRP) and transmits country Information Elements (IEs) in beacons.
* **Airtime Optimization via Rate Pruning (`basic_rate` / `supported_rates`):** On 5 GHz, legacy low data rates (<12 Mbps) are disabled, eliminating slow management frame overhead and encouraging sticky clients to disassociate and roam when their signal strength drops. On the 2.4 GHz radio, standard rates are preserved to ensure compatibility with IoT devices.
* **Native Port Bridging via DSA:** Takes advantage of the RT3200's MediaTek MT7531 switch and the Linux DSA architecture. Normally in AP mode, the WAN port is unusable because routing is disabled. To enable the router's use as an AP connected via its WAN port, we bridge the physical WAN port directly into `br-lan` (`list ports 'wan'`). This eliminates the WAN/LAN boundary, completely turning the AP device into an unmanaged 5-port gigabit switch. The gateway uplink can then be connected via the WAN port (or any LAN port).

### Benefits

* Wi-Fi calling maintains active calls when roaming between APs on both iOS and Android.
* VPN services like Tailscale can operate seamlessly across the bridged APs.

---

## Setup Guide

This guide assumes you have a working OpenWrt 25.12.5+ installation on your routers. The setup procedure below is based on directly editing the configuration files in `/etc/config/` or copying the pre-configured templates from this repository. No GUI steps or UCI commands are required.

### 1. Required Packages

Install the following packages on the gateway and all APs. These are the main packages that enable client roaming.

```bash
apk update
apk del wpad-basic-mbedtls
apk add wpad-wolfssl dawn
```

* **`wpad-wolfssl`** replaces the stock `wpad-basic-mbedtls` with a full WPAD engine including IEEE 802.11k/v roaming support and WPA3 support with Fast Transition (FT).
* `dawn` provides decentralized AP coordination and client steering.

### 2. Repository Layout

* **`gateway/`**: Reference configuration and modification snippets for the gateway router.
  * `firewall-modify`: Modify the default firewall configuration to add rate limiting to the `Allow-Ping` rule.
  * `network-modify`: Configure the `lan` interface to advertise the gateway as the primary DNS server.
  * `system`: Set the router's `hostname` for remote management.
  * `wireless`: Configure the 2.4 GHz and 5 GHz bands, including primary WPA3 roaming interfaces and the dedicated 2.4 GHz IoT VAP.
* **`ap-1/`**: Reference template directory for access points. For each access point in your network, clone the `ap-1/` directory and modify its settings:
  * `network`: Assign a unique static LAN IP (e.g. `192.168.1.2` for `ap-1`, `.3` for `ap-2`, etc.).
  * `system`: Set the node's hostname (e.g. `ap-1`, `ap-2`, etc.).
  * `wireless`: Set the allocated 2.4 GHz and 5 GHz channels per your Wi-Fi channel plan, maintaining matching SSIDs for both primary and IoT networks across nodes.

### 3. Network Configuration (`/etc/config/network`)

#### Gateway

Edit `/etc/config/network` and ensure the `lan` interface advertises the gateway as the primary DNS server:

```
config interface 'lan'
	...
	list dns '192.168.1.1'
```

#### Access Points

For each AP, copy `ap-1/network`, update the static LAN IP in `network` and copy it to `/etc/config/network` on the AP.

* **WAN port as switch port:** Adding `list ports 'wan'` to `config device 'br-lan'` turns the physical WAN port into a fifth gigabit switch port. The uplink cable from the gateway can then plug directly into the WAN port, leaving all four LAN ports free for other wired devices.
* **Router logic removed:** WAN interfaces, WAN firewall zones and IPv6 assignment (`ip6assign`) are eliminated.
* **AP host DNS resolution:** Explicitly configuring `list dns '192.168.1.1'` ensures that on-router processes (such as `apk` package updates, NTP time sync and Tailscale) resolve hostnames directly through the gateway.

### 4. DHCP & DNS Services (`/etc/config/dhcp`)

#### Gateway

Append any static IP hosts to the default `/etc/config/dhcp` configuration and restart the DNS/DHCP server:

```bash
/etc/init.d/dnsmasq restart
```

#### Access Points

Stop and disable `dnsmasq` and `odhcpd` so that DNS requests and DHCP leases
are handled by the gateway router:

```bash
/etc/init.d/dnsmasq stop
/etc/init.d/dnsmasq disable
/etc/init.d/odhcpd stop
/etc/init.d/odhcpd disable
```

### 5. Firewall Configuration (`/etc/config/firewall`)

#### Gateway

Optionally edit `/etc/config/firewall` to add rate limiting to the `Allow-Ping` rule:

```
option limit       10/sec
option limit_burst 20
```

*Note*: The default firewall config has an open-access "Allow-Ping" rule that is not rate-limited. It is good security practice to rate-limit this rule.

#### Access Points

Stop and disable the firewall service entirely, as all traffic is bridged to the gateway:

```bash
/etc/init.d/firewall stop
/etc/init.d/firewall disable
```

### 6. System & Timezone (`/etc/config/system`)

Edit `/etc/config/system` on each device:

```
option hostname 'gateway'  # or 'ap-1', 'ap-2', etc.
option timezone 'PST8PDT,M3.2.0,M11.1.0'
option zonename 'America/Los_Angeles'
```

* Delete the `led_wan` block on the APs because the router functionality is on the gateway.
* *Note:* `musl` typically avoids bundling the full IANA time zone database (`tzdata`) to save flash storage. So an explicit POSIX timezone string is required by `musl` alongside the human-readable zonename.

### 7. Wireless Configuration (`/etc/config/wireless`)

The changes described below are contained in the example `/etc/config/wireless` file in the rep.

#### Primary Fast Roaming Network (802.11k/v)

Add 802.11k and 802.11v options to each primary `wifi-iface` block (`default_radio0` and `default_radio1`):

```
option ieee80211k '1'
option ieee80211v '1'
option bss_transition '1'
```

#### Security & Credentials

Use WPA3-Personal (SAE only) with identical SSID and passphrase values across all nodes:

```
option encryption 'sae'
option ssid '<ExampleNet>'
option key '<ExampleWiFiPassword>'
```

* **SAE Only:** SAE avoids mixed/transitional WPA2/WPA3 (`sae-psk`) modes.
* `option encryption 'sae'` avoids specifying explicit cipher suites such as `ccmp` or `gcmp` (e.g., `sae+ccmp` or `sae+gcmp`). This defers cipher negotiation to `hostapd`.

#### Rate Optimization

On 5 GHz, prune lower legacy data rates to minimize management frame overhead and encourage earlier client roaming. On 2.4 GHz, rate pruning is disabled (commented out) so default driver rates include all standard 802.11b/g rates for maximum IoT compatibility:

```
# 2.4 GHz (radio0)
# default driver rates include all standard 802.11b/g rates
# for compatibility with IoT devices
# option basic_rate '12000 18000 24000'
# option supported_rates '12000 18000 24000 36000 48000 54000'

# 5 GHz (radio1)
option basic_rate '12000 24000'
option supported_rates '12000 24000 36000 48000 54000'
```

#### Dedicated IoT Network (VAP)

Many IoT devices (e.g. sensors, smart home appliances and microcontrollers) do not support WPA3-Personal (SAE), are 2.4 GHz-only, and often fail when connecting to mixed WPA2/WPA3 transition mode networks.

To support these devices without compromising the security or roaming efficiency of the primary network, a dedicated 2.4 GHz Virtual Access Point (VAP) is configured on `radio0`:

```
config wifi-iface 'iot_radio0'
	option device 'radio0'
	option network 'lan'
	option mode 'ap'
	option ssid 'ExampleNet-IoT'
	option encryption 'psk2'
	option key 'ExampleIoTPassword'
	option isolate '1'
```

* **WPA2-PSK (`psk2`):** Ensures compatibility with legacy 2.4 GHz IoT hardware while keeping the primary network pure WPA3-SAE.
* **Client Isolation (`isolate '1'`):** Prevents wireless IoT devices from communicating directly with each other at Layer 2 on the Wi-Fi interface.
* **Stationary by Design:** Fast roaming options (`ieee80211k`, `ieee80211v`, `bss_transition`) and DAWN steering are omitted. IoT devices are stationary, and their firmware often disconnects or misbehaves when receiving 802.11k/v transition frames.

#### Channel Planning

* Configure the 5 GHz channels according to [5G Channel Plan](#5g-channel-plan).
* For 2.4 GHz, assign non-overlapping channels (1, 6, 11) across nodes to avoid co-channel interference.

### 8. Reboot

Reboot the gateway and all access points to ensure clean service initialization.

---

## Operating Guide

### Verifying DAWN

DAWN works out of the box with its default settings:
* **Peer Discovery:** Uses `network_option '2'` (TCP with `umdns` multicast DNS) across the bridged LAN.
* **Shared Key:** Unnecessary and disabled by default.

#### Verification Commands

Run these commands on any node to verify that DAWN is active and communicating:

1. **Check connected DAWN nodes:**
   ```bash
   ubus call dawn get_network
   ```
   *Lists all APs participating in the steering cluster.*

2. **Inspect the hearing map:**
   ```bash
   ubus call dawn get_hearing_map
   ```
   *Displays connected clients and how strongly each AP hears them (RSSI).*

3. **Monitor roaming events in real time:**
   ```bash
   logread -f -e dawn
   ```
   *Shows BSS transition management steering requests when clients move.*

### 5G Channel Plan

In OpenWrt Wi-Fi configurations, `htmode` defines the IEEE wireless standard (802.11ax, etc.) and the channel bandwidth (potentially using bonded channels):
* **HE80 (80 MHz):** Highest throughput (~500+ Mbps), bonding four 20 MHz channels. Maximizes speed where wide, non-overlapping channels are available.
* **HE40 (40 MHz):** Balanced throughput (>200 Mbps), bonding two 20 MHz channels. A good compromise between speed and channel availability in crowded RF environments.
* **HE20 (20 MHz):** Baseline throughput (~100+ Mbps), using a single unbonded 20 MHz channel. Maximizes range, reliability, and channel reuse in dense deployments or high-interference areas.

#### Spectrum & DFS Avoidance

The 5 GHz spectrum is divided into:
* **U-NII-1:** Indoor (channels 36–48)
* **U-NII-2A & U-NII-2C:** DFS bands (avoid)
* **U-NII-3:** High-power spectrum (channels 149–165)

The Belkin RT3200's MediaTek MT7915 radio subsystem has sensitive radar detection circuitry. In residential environments, it frequently detects false positive radar events on DFS channels, causing the router to suddenly drop clients and conduct a mandatory 60-second Channel Availability Check (CAC). To guarantee uninterrupted service, we avoid DFS entirely. This leaves two non-DFS 80 MHz blocks:

* **Block 42:** channels 36 + 40 + 44 + 48 (center channel 42, base channel 36)
* **Block 155:** channels 149 + 153 + 157 + 161 (center channel 155, base channel 149)

#### Example 3-Node Channel Plan

| Device | Role | Usage | Configuration | 5 GHz Block / Channel | 2.4 GHz Channel |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **Gateway** | Gateway | High | HE80 | ch 149 (Block 155) | 11 |
| **AP 1** | AP | High | HE80 | ch 36 (Block 42) | 1 |
| **AP 2** | AP | High | HE20 | ch 165 | 6 |

*Note*: Channel 165 is the most underused channel in the standard 5 GHz spectrum.

* **2.4 GHz separation:** Alternating channels 1, 6 and 11 avoids overlap and eliminates 2.4 GHz co-channel interference.

### Wi-Fi & LAN Bandwidth Benchmarks

* **Server:** LAN PC / Server
  ```bash
  iperf3 -s
  ```
* **Client:** Laptop / Workstation
  ```bash
  iperf3 -c <server-ip-or-host>
  ```
* **Observed Performance:**
  * **Local Wi-Fi (HE80):** ~480 Mbps / 60 MB/sec.
  * **WAN Fiber:** Full line rate (e.g. 93/67 Mbps on 100M plan).

### macOS Local Network Privacy & LAN Access

On macOS (macOS 15+ Sequoia), you may be able to connect via HTTP to an AP's LuCI web interface, but find that `ping` or `ssh` from the terminal fails.

Web browsers automatically prompt for macOS local network permissions, but terminal emulators (e.g. Terminal, WezTerm or iTerm2) often require manual approval:

1. Open **System Settings** on your Mac.
2. Navigate to **Privacy & Security** > **Local Network**.
3. Locate your terminal application (e.g., WezTerm) and toggle the switch to **ON**.

---

## References

* [OpenWrt Hardware: Linksys E8450](https://openwrt.org/toh/linksys/e8450)
* [OpenWrt Wireless Configuration](https://openwrt.org/docs/guide-user/network/wifi/basic)
* [Channel Planning Best Practices](https://www.ekahau.com/blog/channel-planning-best-practices-for-better-wi-fi/)
* [List of WLAN channels](https://en.wikipedia.org/wiki/List_of_WLAN_channels) (Wikipedia)
* [DAWN Documentation](https://github.com/berlin-open-wireless-lab/DAWN)
