# ESP32 Evil Twin + Deauthentication + Captive Portal (Rebuilt)

An educational WiFi security lab built on ESP32 (Arduino core 3.x). Three
modules work together:

1. **Deauthentication** - raw 802.11 deauth/disassoc frames sent from the
   ESP32's radio push clients off a target access point.
2. **Evil Twin** - the ESP32 clones the target's SSID on the same channel;
   kicked clients may reconnect to the clone instead of the real AP.
3. **Captive Portal** - a lure page ("router firmware update failed, verify
   your password") is served to anyone joining the clone. Passwords typed
   into the form are logged and persisted in flash.

For education and authorized testing only. Interfering with networks you do
not own or have written permission to test is illegal in most jurisdictions.
Demonstrate on your own hotspot or a dedicated lab SSID.

---

## 1. How it works

### Stage 0 - control hotspot

After boot the ESP32 exposes an access point:

- SSID: `WiPhi_34732`
- Password: `d347h320`
- Management panel: `http://192.168.4.1/admin`

Every 15 seconds the board scans the 2.4 GHz band and lists nearby networks
(SSID, BSSID, channel) in the panel. Scanning pauses while deauthing, because
a scan would switch channels. Selecting one with `Select` locks it in as the
attack target. `Start deauthing` and `Start EvilTwin` are only accepted once a
target is selected.

The captive portal (lure page) is served on `/` and on any other path (DNS
wildcard) only while the evil twin is running. Otherwise `/` shows the panel.

### Stage 1 - deauthentication

`Start deauthing` switches the radio to the target's channel and pushes
26-byte handcrafted management frames through `esp_wifi_80211_tx()`
(promiscuous mode is enabled before every burst, as raw TX requires it):

- frame control `0xC0` (deauth) plus `0xA0` (disassociation),
- receiver address = broadcast (`FF:FF:FF:FF:FF:FF`),
- transmitter/BSSID = the target's real BSSID,
- reason code 1.

Send rate: one deauth + one disassoc burst (5 repetitions each) every
100 ms. Clients receive an "AP kicked you" frame and drop the link, then
keep retrying - while the evil twin stands by on the same channel.

### Stage 2 - evil twin + captive portal

`Start EvilTwin` reconfigures the built-in AP to broadcast the target SSID
on the target channel. This matters: a clone on the wrong channel either
would not be visible to clients, or would conflict with the deauth bursts.

DNS answers are wildcarded by `DNSServer` (every domain resolves to
`192.168.4.1`), so virtually any HTTP request from a connected device lands
on the lure page. The page posts the entered password back to `/`; the ESP32
then attempts `WiFi.begin()` with the victim network plus that password to
verify it.

### Stage 3 - credential capture

Every submission is logged through `logAttempt()`:

- displayed on `/admin` as `SSID -> password (age)`,
- printed to the serial monitor as `[CAPTURED] ssid -> password`,
- stored in NVS flash (`Preferences`, namespace `etwin`, one blob, cap
  approx. 2500 bytes / about 35 entries; the oldest lines are evicted
  first).

Only well-formed WPA passphrases are logged: 8-63 printable ASCII
characters. Anything else is rejected with a message and not recorded. All
SSIDs and passwords are HTML-escaped before they are shown on the panel.

Entries survive reboots. They are purged automatically after 24 hours of
accumulated board uptime (the uptime counter is also stored in NVS, so no
RTC or NTP is required). The `Delete log` button on `/admin` clears the log
at once.

While a password is being verified, the deauth loop is paused (`verifying`
flag) - otherwise the ESP32 would deauthenticate its own checking connection.
The pause ends when the victim's browser reaches `/result`, or after 30 s if
it never does. If the password is correct, the board joins the victim network,
deauthing stops, the success line is shown on `/admin`, and the control
hotspot is restored.

---

## 2. Admin access during an attack

Important workflow detail:

- While the evil twin is running, the built-in AP carries the **victim's
  SSID**, and `/` serves the lure page. Phones auto-open the lure page on
  connect, which hides the panel.
- The management panel is still reachable at **`http://192.168.4.1/admin`**
  on the same network - open this address manually in a browser.
- The network list is frozen while the evil twin is live (scanning is
  suspended so it does not stall the captive portal).
- `Stop EvilTwin` restores the `WiPhi_34732` control hotspot immediately,
  without a reboot. A power cycle is the fallback if you lose track of the
  panel (for example, if the phone bounced between networks).
- Captured credentials are safe either way: they are held in NVS flash and
  show up on `/admin` after any reboot.

---

## 3. Differences from the original code

The starting point was
[aadesh0706/IOT-ESP32-Evil-Twin-WiFi-Hacking-Deauthentication-Captive-Portal](https://github.com/aadesh0706/IOT-ESP32-Evil-Twin-WiFi-Hacking-Deauthentication-Captive-Portal)
(MIT). That code shipped with the packet-transmission calls commented out, so
the attack could not actually fire, and the example README referenced an
`html/` directory that did not exist in the repository (the portal is
embedded as strings in the sketch). Key changes in this rebuild:

- functional deauth via `esp_wifi_80211_tx()` bursts (the legacy
  `wifi_send_pkt_freedom` approach does not work reliably on core 3.x);
- evil twin AP now follows the target's channel instead of staying on 1;
- all password attempts (including rejected ones) are logged - previously
  only a successful verification was shown;
- log persistence in NVS, 24-hour TTL based on accumulated uptime, manual
  reset button on the panel;
- deauth paused during password verification so correct passwords validate;
- `WiFi.persistent(false)`: credentials captured during verification are no
  longer retained by the SDK, which previously caused the board to
  auto-connect to the victim network after a reboot and break the AP;
- scanning does not block the portal while the evil twin is live, and a
  first scan now runs at boot;
- `index()` renamed (conflicted with the libc function of the same name).

The whole program lives in a single sketch, `NetworkDeAuth.ino`: captive
portal HTML/CSS is generated with string functions (`header()`, `footer()`,
`indexPage()`, the `_tempHTML` table template) rather than served from a
filesystem.

---

## 4. Building and flashing

Toolchain: Arduino IDE with ESP32 core 3.x, or `arduino-cli`:

```bash
arduino-cli core update-index
arduino-cli core install esp32:esp32
arduino-cli compile --fqbn esp32:esp32:esp32 <sketch-folder>
arduino-cli upload -p /dev/ttyUSB0 --fqbn esp32:esp32:esp32
```

On Linux the serial port typically needs one of:

```bash
sudo chmod 666 /dev/ttyUSB0        # one-shot, resets next replug
sudo usermod -aG uucp $USER        # permanent (re-login required)
```

The board can run from a phone power bank - no serial connection is needed
during the demonstration.

## 5. Demo procedure (own network only)

1. Join `WiPhi_34732` (password `d347h320`) and open
   `http://192.168.4.1/admin`.
2. Select a network you own (for example, your phone's hotspot).
3. `Start deauthing` - clients drop the real AP.
4. `Start EvilTwin` - the clone appears on the same channel.
5. Any device that "verifies its password" sends it to the board; refresh
   `/admin` to see the list, or watch the serial monitor.
6. `Stop EvilTwin` and `Stop deauthing` revert everything; `Delete log`
   clears the capture list.

## 6. Known limitations

- The panel and log have no authentication; this is a lab device, not a
  production appliance.
- The 24-hour TTL counts board uptime, not wall-clock time, since there is
  no battery-backed RTC on the board.
- WPA3-only targets and enterprise (802.1X) networks are out of scope.
- Scan results are capped at 16 networks.

## 7. License

MIT. The original project and this rebuild are strictly for educational and
authorized testing purposes.
