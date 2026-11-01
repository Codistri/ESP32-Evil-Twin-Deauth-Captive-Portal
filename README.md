# 🛰️ ESP32 Evil Twin + Deauth + Captive Portal (Own Build)

Educational WiFi-security lab built on an **ESP32**: a deauthentication module
that knocks clients off a target network, an **Evil Twin** access point that
mimics that network, and a **captive portal** that lures victims into
"verifying" their WiFi password. Every entered password is captured into a
persistent log inside the ESP32 flash.

> ⚠️ **Disclaimer** — this project exists for **education and authorized
> testing only**. Running it against networks you do not own or do not have
> written permission to test is illegal almost everywhere (unauthorized
> access / interference with communications equipment). Demo it on your own
> hotspot or lab SSID.

---

## 🧠 How the attack works (3 stages)

```
   [0] SETUP                       [1] DEAUTH                      [2] EVIL TWIN
┌──────────────┐            ┌────────────────────┐          ┌──────────────────────┐
│ ESP32 boots  │            │ ESP32 sends deauth │  phone   │ ESP32 clones the SSID│
│ AP WiPhi_*   │  ─select─▶ │ frames on victim's │ ─kicked─▶│ on the same channel  │
│ admin page   │            │ channel, broadcast │  from AP │ + captive portal     │
└──────────────┘            └────────────────────┘          └──────────┬───────────┘
                                                    victim enters  │  trust
                                                    "router failed │  buttons
                                                     update" page  ▼
                                                 ┌─────────────────────────────┐
                                                 │  password logged → NVS flash│
                                                 │  visible on /admin          │
                                                 └─────────────────────────────┘
```

### [0] Control hotspot — `WiPhi_34732`
The ESP32 boots as an access point (`WiPhi_34732` / `d347h320`). Its web panel
lives at **`http://192.168.4.1/admin`** (no auth — it's a lab device). Every
15 seconds it scans the 2.4 GHz band and shows a table of nearby networks:
SSID, BSSID, channel. You pick one with **Select**.

### [1] Deauthentication (802.11 management frames)
With **Start deauthing**, the firmware hand-crafts raw 802.11 frames
(26 bytes): frame control `0xC0` (deauth) + `0xA0` (disassoc), receiver =
broadcast `FF:FF:FF:FF:FF:FF`, source/BSSID = the victim's AP. Bursts are
pushed through the official ESP-IDF API **`esp_wifi_80211_tx()`** (raw frames
require promiscuous mode enabled), with the radio pinned to the victim's
channel. Legitimate clients see a "you got kicked by the AP" frame and drop
the link; phones retry every few seconds — meanwhile the evil twin is standing
by on the same channel (we spoof authentic MACs, no extra hardware).

### [2] Evil Twin + captive portal
**Start EvilTwin** reconfigures the ESP32's AP: `_selectedNetwork.ssid` +
**the same channel** as the target (crucial — otherwise victims scan past it
and our own deauth frames never land). Any wifi client connecting to it
receives a DNS wildcard answer (`DNSServer`, every domain → `192.168.4.1`) and
a *scary-fake* portal: **"Firmware Update Failed — verify your password."**
Password is POSTed to `/`, the ESP32 itself attempts `WiFi.begin()` with
the victim's network + typed password to validate it.

### [3] Credential capture
Each attempt goes through `logAttempt()`:
- shown instantly on `/admin` — `SSID → password (Nm ago)`;
- printed to serial as `[CAPTURED] ssid -> password`;
- **persisted to NVS flash** (`Preferences`, namespace `etwin`, blob ≤ 2500
  bytes / ~35 entries, oldest evicted) — survives reboots;
- entries are auto-purged after **24 h of board uptime** (the uptime counter
  also lives in NVS, no RTC/NTP needed);
- the **Delete log** button on `/admin` (`/clear`) wipes everything at once.

If the typed password is *correct*, the ESP32 actually joins the victim's
network, flashes "Successfully got password…" on `/admin`, and restores the
control hotspot. While a password is being checked, deauth bursts are paused
(`verifying` flag) so the ESP32 doesn't deauth **itself** out of the
connection it's validating.

---

## 🆚 What we changed vs. the original code

| Original                              | This build                                                          |
| ------------------------------------- | ------------------------------------------------------------------- |
| deauth sending commented out (dead)   | works via `esp_wifi_80211_tx()` bursts, ~10 frames / 100 ms          |
| evil twin forced to channel 1         | AP channel synced to the victim's channel                           |
| "wrong" attempts not logged at all    | every attempt logged (screen, serial, flash)                        |
| log lost on reboot                    | NVS persistence + 24 h TTL + manual reset button                    |
| deauth ran during password verify     | paused while checking — correct passwords now validate reliably     |
| STA creds auto-saved by the SDK       | `WiFi.persistent(false)` — reboot can't hijack the AP into STA mode |
| periodic scan froze the portal        | scanning skipped while the evil twin is live; immediate scan on boot |

---

## 🚀 Setup

1. **Hardware**: any ESP32 devkit + micro-USB data cable (or plain 5 V power
   bank — the board only needs power to run).
2. **Toolchain**: Arduino IDE with the ESP32 core 3.x, or `arduino-cli`:

   ```bash
   arduino-cli core update-index
   arduino-cli core install esp32:esp32
   arduino-cli compile --fqbn esp32:esp32:esp32 <sketch folder>
   arduino-cli upload -p /dev/ttyUSB0 --fqbn esp32:esp32:esp32
   ```

   On Linux, port access usually needs one of:

   ```bash
   sudo chmod 666 /dev/ttyUSB0        # one-shot
   sudo usermod -aG uucp $USER        # permanent, re-login
   ```

3. On first run the serial monitor (115200 baud) is your debug console.

## 📡 Running the demo (own AP only!)

1. Phone/PC → join `WiPhi_34732`, password `d347h320`.
2. Browser → `http://192.168.4.1/admin`.
3. Pick **Select** on the network you own (e.g. your phone's hotspot).
4. **Start deauthing** — clients drop the real AP.
5. **Start EvilTwin** — the fake clone appears on the same channel.
6. Any client that "verifies the password" hands it over:
   see it live on `/admin` (F5) or in the serial monitor.
7. **Stop EvilTwin / Stop deauthing** reverts everything; **Delete log**
   clears the capture list.

---

## 👨‍💻 Credits & license

The starting point was the public repo
[IOT-ESP32-Evil-Twin-WiFi-Hacking-Deauthentication-Captive-Portal](https://github.com/aadesh0706/IOT-ESP32-Evil-Twin-WiFi-Hacking-Deauthentication-Captive-Portal)
by `aadesh0706` (MIT). The deauth/evil-twin logic here is rebuilt on top of
it (deauth transmission, channel sync, credential persistence, admin/log UX)
— see the change table above. Released under MIT, kept the same spirit:
**educational use only.**
