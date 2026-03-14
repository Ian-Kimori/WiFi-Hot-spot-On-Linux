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
sudo systemctl enable hotspot-network.service
```

Now, IP forwarding + NAT rules are applied **automatically at boot**.

---

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

---

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
Profile saved to /etc/NetworkManager/system-connections/pop-hotspot.nmconnection
        │
        ▼
Every subsequent run (script)
  nmcli connection up pop-hotspot  ──► reads saved profile ──► starts hotspot
```

The values (`ssid`, `password`) only need to be passed **once** — when creating the profile.
After that, `nmcli connection up pop-hotspot` knows exactly what to do. 
Your script is fine as-is; just make sure you've run the manual command at least once beforehand to create the profile.
---

## **Summary of the final setup**

| Component                   | Purpose                                                          |
| --------------------------- | ---------------------------------------------------------------- |
| `wifi-hotspot.sh`          | Sets up IP forwarding & NAT automatically                           |
| `hotspot.service`   | Runs the script at boot, so rules are applied automatically      |
| `nmcli device wifi hotspot` | Manual hotspot start — lets you choose SSID & password each time |
