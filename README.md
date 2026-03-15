# WiFi-Hot-spot-On-Linux

---

## **Phase 1: Checking your Ethernet and Wi-Fi capabilities**

1. **Check supported modes of your Wi-Fi adapter**:

```bash
iw list | grep -A 10 "Supported interface modes"
```

* We confirmed that your Wi-Fi adapter (`wlp0s20f3`) supports **AP mode**, which is required to create a hotspot.

2. **Check devices and connections**:

```bash
nmcli device status
```

* Found:

  * Ethernet (`enp0s31f6`) → connected (provides internet from ONU)
  * Wi-Fi (`wlp0s20f3`) → disconnected (available for hotspot)

---

## **Phase 2: Testing a temporary hotspot**

We first try to start a **quick manual hotspot** to see if it works:

```bash
nmcli device wifi hotspot ifname wlp0s20f3 con-name pop-hotspot ssid PopOS-Hotspot password StrongPass123
```

* This successfully activates a hotspot.
* Limitation: **not persistent**, no NAT rules applied yet. Devices could connect but don’t have internet.

---

## **Phase 3: Enable IP forwarding & NAT (internet sharing)**

1. **Check IP forwarding**:

```bash
cat /proc/sys/net/ipv4/ip_forward
```

* Value `1` means forwarding is enabled; if `0`, you would enable it with:

```bash
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
```

2. **Add NAT rules for sharing Ethernet to hotspot**:

```bash
sudo iptables -t nat -A POSTROUTING -s 10.42.0.0/24 -o enp0s31f6 -j MASQUERADE
sudo iptables -A FORWARD -i wlp0s20f3 -o enp0s31f6 -j ACCEPT
sudo iptables -A FORWARD -i enp0s31f6 -o wlp0s20f3 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

* This allow devices connected to the hotspot to access your Ethernet internet.

3. **Test the connection** → phone could see the hotspot but initially said “connected, no internet” because some rules or MASQUERADE weren’t applied correctly. Fix by ensuring correct POSTROUTING and forwarding.

---

## **Phase 4: Making IP forwarding & NAT persistent**

We now create a **script** to set up IP forwarding & NAT automatically:

```bash
sudo nano /usr/local/bin/wifi-hotspot.sh
```

```bash
#!/bin/bash
# Enable IP forwarding
echo 1 > /proc/sys/net/ipv4/ip_forward

# NAT hotspot subnet to Ethernet
iptables -t nat -A POSTROUTING -s 10.42.0.0/24 -o enp0s31f6 -j MASQUERADE

# Allow forwarding
iptables -A FORWARD -i wlp0s20f3 -o enp0s31f6 -j ACCEPT
iptables -A FORWARD -i enp0s31f6 -o wlp0s20f3 -m state --state RELATED,ESTABLISHED -j ACCEPT

# Start the hotspot
#nmcli connection up pop-hotspot
```

* Make it executable:

```bash
sudo chmod +x /usr/local/bin/wifi-hotspot.sh
```

* We also create a **systemd service** to run this script at boot so IP forwarding & NAT are **automatic**, without having to type anything.
 
* Using **nano** we create a **systemd service**

```bash
sudo nano /etc/systemd/system/hotspot.service
```

```ini
[Unit]
Description=Set up NAT & forwarding for Wi-Fi hotspot
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/wifi-hotspot.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

* Enable the service:

```bash
sudo systemctl daemon-reload
```

```bash
sudo systemctl enable hotspot.service
```
Now, IP forwarding + NAT rules are applied **automatically at boot**.

---
Breaking down every line:

---

## `#!/bin/bash`
Called a **shebang**. Tells the OS to run this file using the bash interpreter. Must be the first line of any bash script.

---

## `echo 1 > /proc/sys/net/ipv4/ip_forward`
- `/proc/sys/net/ipv4/ip_forward` is a **kernel parameter file** — writing `1` to it enables **IP forwarding**
- IP forwarding allows your machine to **pass packets between interfaces** (WiFi → Ethernet and back)
- Without this, your machine would receive packets from hotspot clients but refuse to forward them to the internet
- This change is **temporary** — resets to `0` on reboot

---

## `iptables -t nat -A POSTROUTING -s 10.42.0.0/24 -o enp0s31f6 -j MASQUERADE`

| Token | Meaning |
|---|---|
| `iptables` | Tool for configuring Linux kernel firewall/packet filtering rules |
| `-t nat` | Work on the **NAT table** (Network Address Translation) |
| `-A POSTROUTING` | **Append** a rule to the POSTROUTING chain — runs after routing decisions are made, just before packet leaves |
| `-s 10.42.0.0/24` | Match packets whose **source IP** is in the hotspot subnet (IPs assigned to connected devices) |
| `-o enp0s31f6` | Match packets going **out through** your Ethernet interface |
| `-j MASQUERADE` | **Disguise** the source IP — replace the client's private IP (e.g. `10.42.0.5`) with your machine's public Ethernet IP so the internet can reply back |

In plain terms: **"Any packet from a hotspot client heading to the internet — swap its source IP with mine so responses come back to me, then I'll forward them back to the client."**

---

## `iptables -A FORWARD -i wlp0s20f3 -o enp0s31f6 -j ACCEPT`

| Token | Meaning |
|---|---|
| `-A FORWARD` | Append to the **FORWARD chain** — handles packets passing *through* your machine (not destined for it) |
| `-i wlp0s20f3` | Packets coming **in** from your WiFi hotspot interface |
| `-o enp0s31f6` | Going **out** through your Ethernet interface |
| `-j ACCEPT` | **Allow** them through |

In plain terms: **"Allow hotspot clients to send traffic out to the internet via Ethernet."** This is the outbound direction.

---

## `iptables -A FORWARD -i enp0s31f6 -o wlp0s20f3 -m state --state RELATED,ESTABLISHED -j ACCEPT`

| Token | Meaning |
|---|---|
| `-i enp0s31f6` | Packets coming **in** from Ethernet (internet responses) |
| `-o wlp0s20f3` | Going **out** to your WiFi hotspot clients |
| `-m state` | Load the **state tracking module** — lets iptables track connection state |
| `--state RELATED,ESTABLISHED` | Only allow packets that belong to an **already established** connection or are related to one (e.g. FTP data channel). Blocks unsolicited inbound packets. |
| `-j ACCEPT` | Allow them through |

In plain terms: **"Allow internet responses to flow back to hotspot clients — but only if they asked for it first."** This is the inbound direction with a security filter.

---

## `nmcli connection up pop-hotspot`
- `nmcli connection up` — activates a saved NetworkManager connection profile
- `pop-hotspot` — the profile name you created earlier
- This **starts the WiFi hotspot** using the saved SSID, password, and interface settings

Currently commented out with `#` so it won't run — you'd uncomment it once the profile is confirmed persistent in `/etc`.

---

## The full picture — packet flow:

```
Phone/laptop
    │
    │  connects via WiFi (wlp0s20f3)
    ▼
Your Pop!_OS machine
    │
    │  IP forwarding enabled ──► iptables FORWARD rules allow it
    │  MASQUERADE hides client IP
    ▼
Ethernet (enp0s31f6)
    │
    ▼
Internet
    │
    │  response comes back to your Ethernet IP
    │  iptables allows ESTABLISHED/RELATED back through
    ▼
Your Pop!_OS machine
    │
    ▼
WiFi client gets the response
```
## **Phase 5: Manual hotspot start (final setup)**

* Because you want to **choose hotspot name & password every time**, we **don’t put `nmcli` in the script**.

* To start the hotspot manually after boot:

```bash
nmcli device wifi hotspot ifname wlp0s20f3 con-name pop-hotspot ssid "MyHotspotName" password "MyHotspotPass"
```

* You can change the `ssid` and `password` each time.

* Your phone connects and gets internet immediately, because IP forwarding & NAT are already active.

* To stop the hotspot:

```bash
nmcli connection down pop-hotspot
```
---

Here's what's happening in each case:

---

## When you run the script (automatic)

```bash
nmcli connection up pop-hotspot
```

This **brings up an existing saved connection profile** named `pop-hotspot`. 

The SSID, password, channel, band — everything — was already saved into that profile the **first time** you ran the manual command. nmcli just reads from that stored profile.

You can inspect what's saved with:
```bash
nmcli connection show pop-hotspot
```

---

## When you run the manual command

```bash
nmcli device wifi hotspot ifname wlp0s20f3 con-name pop-hotspot ssid "MyHotspotName" password "MyHotspotPass"
```

This does **two things at once:**
1. **Creates or overwrites** the connection profile named `pop-hotspot` with the SSID and password you specified
2. **Immediately starts** the hotspot

So the first time you run this, it *saves* those values. Every subsequent `nmcli connection up pop-hotspot` just reuses them.

## The flow in plain terms

```
First run (manual command)
        │
        ▼
Creates profile "pop-hotspot"
  - SSID: MyHotspotName
  - Password: MyHotspotPass
  - Interface: wlp0s20f3
        │
        ▼
Profile saved to /run/NetworkManager/system-connections/pop-hotspot.nmconnection or
Profile saved to /etc/NetworkManager/system-connections/pop-hotspot.nmconnection
        │
        ▼
Every subsequent run (script)
  nmcli connection up pop-hotspot  ──► reads saved profile ──► starts hotspot
```

---

## Key takeaway

The values (`ssid`, `password`) only need to be passed **once** — when creating the profile. After that, `nmcli connection up pop-hotspot` knows exactly what to do. Your script is fine as-is; just make sure you've run the manual command at least once beforehand to create the profile.

---
Let's find it. Run these one by one:

---

**1. Check if the profile exists at all in NetworkManager:**
```bash
nmcli connection show
```
Look for `pop-hotspot` in the list. If it's there, it exists somewhere.

---

**2. Find where exactly it's saved:**
```bash
sudo find / -name "pop-hotspot*" 2>/dev/null
```

---

**3. Check if it's in the runtime directory instead:**
```bash
ls /run/NetworkManager/system-connections/
```
Some distros/versions save connections here temporarily.

---

**4. Check if NetworkManager is using a different config directory:**
```bash
sudo NetworkManager --print-config | grep paths
```
or
```bash
cat /etc/NetworkManager/NetworkManager.conf
```
Look for a `[keyfile]` section with a custom `path=` — it might be pointing somewhere non-standard.

---

## Most likely explanations if it's missing:

| Reason | What happened |
|---|---|
| Profile was never saved | The `nmcli device wifi hotspot` command created a **temporary** session only |
| Custom NM config | NetworkManager is storing profiles in a non-default directory |
| Profile was deleted | Something wiped it (system update, `nmcli connection delete`) |

Found it. Look at this line:

```
netplan-NM-6191eb96-1f46-45a1-bb57-a1caf4dfecc6-PopOS-Hotspot.nmconnection
```

It's in `/run/NetworkManager/system-connections/` not `/etc/NetworkManager/system-connections/`. Read it with:

```bash
sudo cat "/run/NetworkManager/system-connections/netplan-NM-6191eb96-1f46-45a1-bb57-a1caf4dfecc6-PopOS-Hotspot.nmconnection"
```

---

## Why is it there and not in `/etc`?

Your system is using **Netplan** as the network configuration layer on top of NetworkManager. Netplan manages connections and stores its generated NM profiles in `/run/NetworkManager/system-connections/` (which is **temporary/volatile** — it gets wiped on reboot) instead of the persistent `/etc/NetworkManager/system-connections/`.

---

## This explains a lot — `/run` is a RAM-based tmpfs:

```
/run  →  lives in memory  →  cleared on every reboot
/etc  →  lives on disk    →  persists across reboots
```

So your hotspot profile **disappears on reboot**, which is exactly why your startup script uses `nmcli connection up pop-hotspot` — it needs the profile to have been created first in that session.

---

## To make it truly persistent, move it to `/etc`:

```bash
sudo cp "/run/NetworkManager/system-connections/netplan-NM-6191eb96-1f46-45a1-bb57-a1caf4dfecc6-PopOS-Hotspot.nmconnection" /etc/NetworkManager/system-connections/pop-hotspot.nmconnection
```

```bash
sudo chmod 600 /etc/NetworkManager/system-connections/pop-hotspot.nmconnection
```

```bash
sudo nmcli connection reload
```

After that, `nmcli connection show` should list `pop-hotspot` sourced from `/etc` and it will survive reboots — making your auto-start script fully reliable.

Breaking it down piece by piece:

---

**`nmcli`**
The command-line tool for NetworkManager. It's how you control network connections, devices, and settings from the terminal without a GUI.

---

**`device wifi`**
Tells nmcli you're working with a **WiFi device** specifically. `device` is the object category, `wifi` narrows it down to wireless operations.

---

**`hotspot`**
The action/subcommand. Tells nmcli to create an **access point (AP)** — i.e., turn your WiFi card into a hotspot that other devices can connect to, rather than connecting to an existing network.

---

**`ifname wlp0s20f3`**
- `ifname` = interface name
- `wlp0s20f3` = your specific **physical WiFi card** identifier on your Pop!_OS machine

This tells nmcli *which* WiFi interface to use for the hotspot, in case you had multiple wireless cards.

---

**`con-name pop-hotspot`**
- `con-name` = connection name
- `pop-hotspot` = the **label/profile name** NetworkManager saves this connection under

This is just a human-readable identifier. It's what you reference later when running `nmcli connection up pop-hotspot`. It has nothing to do with the SSID broadcast name.

---

**`ssid PopOS-Hotspot`**
- `ssid` = Service Set Identifier
- `PopOS-Hotspot` = the **network name that other devices see** when scanning for WiFi

This is what shows up on your phone/laptop when you search for available networks.

---

**`password StrongPass123`**
The **WPA2 passphrase** required for other devices to connect to your hotspot. NetworkManager automatically uses WPA2-Personal security when you set this.

---

## Summary table

| Token | Role | What it affects |
|---|---|---|
| `nmcli` | Tool | — |
| `device wifi` | Category | Scopes to WiFi operations |
| `hotspot` | Action | Creates an AP |
| `ifname wlp0s20f3` | Hardware target | Which physical card to use |
| `con-name pop-hotspot` | Profile label | How NM saves/references it internally |
| `ssid PopOS-Hotspot` | Broadcast name | What other devices see |
| `password StrongPass123` | Auth passphrase | WPA2 key to join the hotspot |

Here's a breakdown of all the actions available under `nmcli device wifi`:

---

## `nmcli device wifi [action]`

| Action | Syntax | What it does |
|---|---|---|
| `list` | `nmcli device wifi list` | Lists all nearby visible WiFi networks (SSIDs, signal strength, security type, channel) |
| `connect` | `nmcli device wifi connect <SSID>` | Connects to an existing WiFi network by SSID |
| `hotspot` | `nmcli device wifi hotspot ...` | Creates an access point (what you've been using) |
| `rescan` | `nmcli device wifi rescan` | Forces a fresh scan for nearby networks |
| `show-password` | `nmcli device wifi show-password` | Shows the password of the currently active WiFi connection as text + QR code in terminal |

---

## But `device wifi` is just one category. `nmcli device` has more:

| Action | What it does |
|---|---|
| `nmcli device status` | Shows all network interfaces and their state (connected, disconnected, unmanaged) |
| `nmcli device show <ifname>` | Detailed info about a specific interface (IP, MAC, MTU, etc.) |
| `nmcli device connect <ifname>` | Activates a device |
| `nmcli device disconnect <ifname>` | Disconnects a device |
| `nmcli device reapply <ifname>` | Reapplies connection settings to a device without fully reconnecting |
| `nmcli device monitor <ifname>` | Watches a device for state changes in real time |
| `nmcli device set <ifname>` | Changes device properties (e.g. managed/unmanaged) |
| `nmcli device delete <ifname>` | Removes a software device (like a bridge or dummy interface) |

---

## And the broader nmcli object categories:

| Object | Purpose |
|---|---|
| `nmcli device` | Manage physical/virtual network interfaces |
| `nmcli connection` | Manage saved connection profiles |
| `nmcli general` | NetworkManager overall status, logging, permissions |
| `nmcli networking` | Turn networking on/off entirely |
| `nmcli radio` | Control WiFi/WWAN radio switches (like rfkill) |
| `nmcli monitor` | Watch all NetworkManager events live |

---

The ones most relevant to your hotspot setup are `nmcli connection` (for managing the saved `pop-hotspot` profile) and `nmcli radio` (which you've used before with rfkill issues on your machine).

## **Summary of the final setup**

| Component                   | Purpose                                                          |
| --------------------------- | ---------------------------------------------------------------- |
| `wifi-hotspot.sh`          | Sets up IP forwarding & NAT automatically                           |
| `hotspot.service`   | Runs the script at boot, so rules are applied automatically      |
| `nmcli device wifi hotspot` | Manual hotspot start — lets you choose SSID & password each time |
