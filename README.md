# 📡 Evil Twin Captive Portal Attack with Airgeddon
### Kali Linux · Panda Wireless PAU0B AC600 · Airgeddon v11.61

> **⚠️ Educational & Authorized Testing Only**  
> This documentation is intended for penetration testers and security researchers operating on networks they own or have explicit written permission to test. Unauthorized use against networks you do not own is illegal.

---

## 📋 Table of Contents

- [Hardware Required](#-hardware-required)
- [Software Required](#-software-required)
- [Attack Overview](#-attack-overview)
- [Step-by-Step Walkthrough](#-step-by-step-walkthrough)
  - [Step 1 — Launch Airgeddon & Select Interface](#step-1--launch-airgeddon--select-interface)
  - [Step 2 — Navigate to Evil Twin Attacks Menu](#step-2--navigate-to-evil-twin-attacks-menu)
  - [Step 3 — Scan for Target Networks](#step-3--scan-for-target-networks)
  - [Step 4 — Select Target Network](#step-4--select-target-network)
  - [Step 5 — Choose Deauthentication Method](#step-5--choose-deauthentication-method)
  - [Step 6 — Disable DoS Pursuit Mode](#step-6--disable-dos-pursuit-mode)
  - [Step 7 — Skip MAC Spoofing & Capture Handshake](#step-7--skip-mac-spoofing--capture-handshake)
  - [Step 8 — Handshake Capture in Progress](#step-8--handshake-capture-in-progress)
  - [Step 9 — Handshake Successfully Captured](#step-9--handshake-successfully-captured)
  - [Step 10 — Configure Captive Portal Language](#step-10--configure-captive-portal-language)
  - [Step 11 — Attack Running (All Windows)](#step-11--attack-running-all-windows)
  - [Step 12 — Password Captured](#step-12--password-captured)
- [How It Works (Technical Summary)](#-how-it-works-technical-summary)
- [Troubleshooting](#-troubleshooting)

---

## 🔧 Hardware Required

| Component | Details |
|---|---|
| **Attack Machine** | Any PC/laptop running Kali Linux |
| **External WiFi Adapter** | **Panda Wireless PAU0B AC600** (dual-band 2.4GHz / 5GHz) |
| **Why external adapter?** | The internal NIC is used for internet/management; the external adapter is dedicated to monitor mode and AP injection |

> The Panda PAU0B is ideal for this attack because it supports **monitor mode**, **packet injection**, and **AP mode** — all required for airgeddon's Evil Twin module. Its dual-band support (2.4GHz & 5GHz) allows targeting networks on either band.

---

## 💻 Software Required

| Software | Purpose |
|---|---|
| **Kali Linux** | Attack OS with all tools pre-installed |
| **Airgeddon v11.61** | All-in-one WiFi attack framework |
| **aircrack-ng suite** | Packet capture, monitor mode (`airmon-ng`) |
| **mdk4** | Deauthentication flood (amok mode) |
| **hostapd** | Creates the rogue Evil Twin access point |
| **dnsmasq** | DHCP server — assigns IPs to victims connecting to the fake AP |
| **lighttpd** | Lightweight web server serving the captive portal page |
| **lighttpd / PHP** | Handles credential form submission |

### Install / Launch Airgeddon

```bash
# Clone airgeddon
git clone https://github.com/v1s1t0r1sh3r3/airgeddon.git
cd airgeddon

# Run as root
sudo bash airgeddon.sh
```

### Enable Monitor Mode (airgeddon handles this, but manually if needed)

```bash
# Kill interfering processes first
sudo airmon-ng check kill

# Put adapter into monitor mode
sudo airmon-ng start wlan1

# Verify — adapter should now appear as wlan1mon
sudo iwconfig
```

---

## 🗺️ Attack Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    EVIL TWIN CAPTIVE PORTAL FLOW                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. PUT ADAPTER IN MONITOR MODE (wlan1 → wlan1mon)             │
│           ↓                                                     │
│  2. SCAN for nearby WPA2 networks                               │
│           ↓                                                     │
│  3. SELECT TARGET (Fios-L4tHF, Ch.1, BSSID B8:F8:53:8B:1E:3A) │
│           ↓                                                     │
│  4. CAPTURE WPA2 HANDSHAKE (forced via mdk4 deauth flood)       │
│           ↓                                                     │
│  5. LAUNCH EVIL TWIN AP (clone of real router)                  │
│     + DEAUTH FLOOD (kick clients off real AP)                   │
│     + DHCP SERVER (assign IPs to victims)                       │
│     + CAPTIVE PORTAL (fake "router firmware update" page)       │
│           ↓                                                     │
│  6. VICTIM connects to fake AP, enters WiFi password            │
│           ↓                                                     │
│  7. AIRGEDDON validates password against captured handshake     │
│           ↓                                                     │
│  8. CORRECT PASSWORD → displayed in Control panel ✓            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📸 Step-by-Step Walkthrough

---

### Step 1 — Launch Airgeddon & Select Interface

```bash
sudo bash airgeddon.sh
```

Airgeddon launches and detects the available wireless interfaces. Select the Panda adapter (`wlan1`) and put it in **Monitor Mode** via option `2`. Once in monitor mode, the interface is renamed to `wlan1mon`.

The main menu displays all available attack categories. For this attack, we'll use option **7 — Evil Twin attacks menu**.

<p align="center">
  <img src="screenshots/1777243734649_3_0_1.png" alt="Airgeddon Main Menu" width="750"/>
</p>

> **Interface selected:** `wlan1mon` | **Mode:** Monitor | **Supported bands:** 2.4GHz, 5GHz

---

### Step 2 — Navigate to Evil Twin Attacks Menu

From the main menu, select `7` to enter the **Evil Twin attacks menu**.

The menu presents several Evil Twin variants:
- **Option 5** — Just the rogue AP, no sniffing (use with external sniffer)
- **Options 6–8** — AP with sniffing / SSL stripping
- **Option 9** — **AP with captive portal** ← *this is what we're using*

We select **option 9 — Evil Twin AP attack with captive portal** because it tricks victims into voluntarily submitting their WiFi password through a fake login/router page, rather than relying on passive sniffing.

<p align="center">
  <img src="screenshots/3_0_2.png" alt="Evil Twin Attacks Menu" width="750"/>
</p>

---

### Step 3 — Scan for Target Networks

Before selecting a specific target, airgeddon opens a **live scan window** using `airodump-ng`. This captures beacon frames from all nearby access points, listing their BSSID, channel, signal strength, encryption type, and ESSID.

Press `Ctrl+C` to stop scanning once you've identified your target.

<p align="center">
  <img src="screenshots/1777243734649_3_0_3.png" alt="Scanning for Target Networks" width="850"/>
</p>

> **What you're seeing in the scan window:**  
> Top section = Access Points (BSSIDs, channels, power, encryption)  
> Bottom section = Associated clients currently connected to those APs  
> Networks marked `*` have active clients — required for a successful deauth + capture attack

---

### Step 4 — Select Target Network

After the scan completes, airgeddon presents a **formatted target list**. Networks marked with `(*)` have detected clients connected to them — these are the best targets since we need to deauthenticate an active client to force a WPA2 handshake.

We select **option 7** — `Fios-L4tHF` on channel 1, BSSID `B8:F8:53:8B:1E:3A`, signal strength 70%.

<p align="center">
  <img src="screenshots/1777243734650_3_0_4.png" alt="Select Target Network" width="750"/>
</p>

| Field | Value |
|---|---|
| BSSID | `B8:F8:53:8B:1E:3A` |
| Channel | `1` |
| Signal | `70%` |
| Encryption | `WPA2` |
| ESSID | `Fios-L4tHF` |
| Status | `(*)` — has active clients |

---

### Step 5 — Choose Deauthentication Method

The **deauth sub-menu** asks which method to use for kicking clients off the legitimate AP. This is essential — we need clients to disconnect from the real router and (hopefully) reconnect to our Evil Twin.

Three options are available:
1. **Deauth / disassoc amok mdk4 attack** ← *selected* — floods the channel with deauth/disassoc frames targeting all clients
2. **Deauth aireplay attack** — targeted deauth using `aireplay-ng`
3. **Auth DoS attack** — authentication request flood

We select **option 1 (mdk4 amok mode)** because it's the most aggressive and effective at clearing clients from the legitimate AP.

<p align="center">
  <img src="screenshots/1777243734650_3_0_5.png" alt="Deauth Method Selection" width="750"/>
</p>

---

### Step 6 — Disable DoS Pursuit Mode

Airgeddon asks if we want to enable **"DoS pursuit mode"** — a feature that would automatically re-launch the deauth attack if the target AP switches channels (channel hopping). This requires a *second* dedicated wireless interface in monitor mode.

Since we only have one external adapter, we answer **N**.

<p align="center">
  <img src="screenshots/1777243734650_3_0_6.png" alt="DoS Pursuit Mode Prompt" width="750"/>
</p>

> **DoS pursuit mode** is useful when attacking enterprise APs that actively change channels to evade jamming. For home routers (like this Verizon Fios router on a fixed channel), it's unnecessary.

---

### Step 7 — Skip MAC Spoofing & Capture Handshake

Two prompts appear in sequence:

**1. MAC Address Spoofing** — Airgeddon asks whether to randomize/spoof the MAC address of our interface during the attack. We answer **N** for this demonstration.

> In a real pentest, spoofing your MAC is good practice to reduce forensic traceability on the target network's logs.

**2. Handshake Capture** — The captive portal attack requires a previously captured **WPA2 4-way handshake** from the target network. Airgeddon uses this to *validate* whatever password the victim enters into the portal — if the password decrypts the handshake, the attack confirms success.

We answer **N** (no existing file) so airgeddon will **capture the handshake now** by forcing a deauth and listening for the reconnect.

<p align="center">
  <img src="screenshots/1777243734650_3_0_8.png" alt="MAC Spoof and Handshake Prompt" width="750"/>
</p>

---

### Step 8 — Handshake Capture in Progress

Airgeddon sets a **20-second timeout** and opens two concurrent windows:
- **Window 1** — `airodump-ng` listening for the WPA2 4-way handshake
- **Window 2** — `mdk4` / `aireplay-ng` sending deauth frames to force clients to reconnect

When a client disconnects and reconnects to the real AP, the handshake exchange is captured automatically.

```bash
# What airgeddon runs internally (simplified):
airodump-ng --bssid B8:F8:53:8B:1E:3A --channel 1 -w /root/handshake wlan1mon &
mdk4 wlan1mon d -B B8:F8:53:8B:1E:3A
```

<p align="center">
  <img src="screenshots/1777243734651_3_0_9.png" alt="Handshake Capture Timeout" width="700"/>
</p>

---

### Step 9 — Handshake Successfully Captured

The handshake capture succeeds within the 20-second window. Airgeddon also reports a **PMKID** was captured as a bonus — PMKID attacks can crack WPA2 without requiring a full 4-way handshake and work against some router implementations.

The handshake `.cap` file is saved to:
```
/root/handshake-B8:F8:53:8B:1E:3A.cap
```

Press Enter to accept the default save path and continue to captive portal configuration.

<p align="center">
  <img src="screenshots/1777243734651_3_1_0.png" alt="Handshake Captured Successfully" width="850"/>
</p>

> **Why do we need the handshake?**  
> The captive portal itself doesn't crack the password — it tricks the victim into *typing it*. The handshake is used purely to **verify** that what the victim typed is actually correct before airgeddon declares success and stops the attack.

---

### Step 10 — Configure Captive Portal Language & Style

**Language Selection** — Airgeddon asks what language to display the captive portal in. We select **1 (English)**.

**Advanced Portal** — We select **Y** for the advanced captive portal. This generates a branded portal page using the target AP's BSSID to look up the router vendor (Verizon/Fios in this case) and inserts their logo — making the fake "router firmware update" page appear much more convincing to the victim.

<p align="center">
  <img src="screenshots/1777243734651_3_1_2.png" alt="Captive Portal Language and Style" width="750"/>
</p>

> **Standard portal** shows a generic WiFi login page.  
> **Advanced portal** shows a vendor-branded page (e.g., "Your Fios router requires a firmware update. Please enter your WiFi password to continue.") — significantly more convincing.

---

### Step 11 — Attack Running (All Windows Active)

The full attack is now live. Airgeddon spawns **5 concurrent windows**:

| Window | Tool | Purpose |
|---|---|---|
| **AP** | `hostapd` | Broadcasts the Evil Twin AP with the cloned SSID |
| **DHCP** | `dnsmasq` | Assigns IP addresses to victims connecting to the fake AP |
| **Deauth** | `mdk4` | Continuously deauthenticates clients from the real AP |
| **DNS** | `dnsmasq` | Intercepts all DNS queries, redirects everything to the captive portal |
| **Webserver** | `lighttpd` | Serves the captive portal HTML/PHP page |
| **Control** | airgeddon | Monitors connection attempts and validates submitted passwords |

<p align="center">
  <img src="screenshots/1777243734651_3_1_3.png" alt="Full Attack Running - All Windows" width="1000"/>
</p>

**What the victim experiences:**
1. Their device is kicked off `Fios-L4tHF` (real AP) by the deauth flood
2. Their device sees `Fios-L4tHF` still broadcasting (our Evil Twin)
3. They connect to our AP (no password required — it's an open AP)
4. Any web request redirects to the captive portal
5. Portal displays a branded "firmware update required" page asking for the WiFi password

---

### Step 12 — Password Captured ✅

The Control panel confirms a victim has connected to the Evil Twin AP (DHCP lease issued to `6a:ec:c4:04:98:33`) and accessed the captive portal. The submitted password is displayed in real time.

Airgeddon automatically **validates the password against the captured handshake** — if it's correct, the attack stops gracefully and the password is confirmed.

<p align="center">
  <img src="screenshots/1777243734652_3_1_4.png" alt="Password Captured in Control Panel" width="800"/>
</p>

```
Attempts: 1  |  Last password: EvilTwinLab  |  Portal accessed ✓
```

> The attack confirms the submitted password is valid, stops the deauth flood, and logs the credential. The victim's device reconnects to the real AP and they likely never know anything happened.

---

## 🔍 How It Works (Technical Summary)

```
REAL AP  ←──── mdk4 deauth flood ────  wlan1mon (monitor)
   ↑                                         ↓
   │                                   hostapd (Evil Twin AP)
   │                                   same SSID, open auth
   │                                         ↓
VICTIM ──── kicked off real AP ────→  connects to Evil Twin
                                             ↓
                                       dnsmasq (DHCP + DNS)
                                       all DNS → 192.168.1.1
                                             ↓
                                       lighttpd captive portal
                                       "Enter WiFi password"
                                             ↓
                              airgeddon validates vs handshake .cap
                                             ↓
                                    ✅ PASSWORD CONFIRMED
```

The core principle exploits the fact that **WPA2-Personal has no mutual authentication** — clients cannot verify the AP is legitimate, only that it knows the PSK. By operating an open (no-password) AP with the same SSID, we bypass this entirely and use the captive portal to socially engineer the credential.

---

## 🛠️ Troubleshooting

| Issue | Solution |
|---|---|
| Interface not showing in airgeddon | Run `sudo airmon-ng check kill` before launching |
| Handshake capture fails | Wait for a connected client; try increasing timeout to 60s |
| No clients connecting to Evil Twin | Signal must be equal or stronger than real AP — get closer |
| Captive portal not loading for victim | Verify `dnsmasq` and `lighttpd` windows show no errors |
| Advanced portal logo not showing | Verify internet access on attacker machine for vendor lookup |
| mdk4 not found | `sudo apt install mdk4` |

> **Additional troubleshooting:** [Airgeddon FAQ & Troubleshooting Wiki](https://github.com/v1s1t0r1sh3r3/airgeddon/wiki/FAQ%20&%20Troubleshooting)

---

*Documented on Kali Linux · Airgeddon v11.61 · Panda Wireless PAU0B AC600*
