# WiFi-Hot-spot-On-Linux

Absolutely! Let’s go **step by step**, from the very start to your **final setup**, so you have the full picture of what we did and how it all fits together. I’ll break it into **phases**: checking, configuring, testing, and making it manual/persistent.

---

## **Phase 1: Checking your Wi-Fi and Ethernet capabilities**

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

We first tried to start a **quick manual hotspot** to see if it works:

```bash
nmcli device wifi hotspot \
ifname wlp0s20f3 \
con-name pop-hotspot \
ssid PopOS-Hotspot \
password StrongPass123
```

* This successfully activated a hotspot.
* Limitation: **not persistent**, no NAT rules applied yet. Devices could connect but didn’t have internet.

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

* This allowed devices connected to the hotspot to access your Ethernet internet.

3. **Tested the connection** → phone could see the hotspot but initially said “connected, no internet” because some rules or MASQUERADE weren’t applied correctly. Fixed by ensuring correct POSTROUTING and forwarding.

---

## **Phase 4: Making NAT & forwarding persistent**

We created a **script** to set up forwarding & NAT automatically:

**`/usr/local/bin/start-hotspot.sh`:**

```bash
#!/bin/bash
# Enable IP forwarding
echo 1 > /proc/sys/net/ipv4/ip_forward

# NAT hotspot subnet to Ethernet
iptables -t nat -A POSTROUTING -s 10.42.0.0/24 -o enp0s31f6 -j MASQUERADE

# Allow forwarding
iptables -A FORWARD -i wlp0s20f3 -o enp0s31f6 -j ACCEPT
iptables -A FORWARD -i enp0s31f6 -o wlp0s20f3 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

* Made it executable:

```bash
sudo chmod +x /usr/local/bin/start-hotspot.sh
```

* We also created a **systemd service** to run this script at boot so NAT & forwarding are **automatic**, without having to type anything.

**`/etc/systemd/system/hotspot-network.service`:**

```ini
[Unit]
Description=Set up NAT & forwarding for Wi-Fi hotspot
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/start-hotspot.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

* Enabled the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable hotspot-network.service
```

✅ Now, NAT + forwarding rules are applied **automatically at boot**.

---

## **Phase 5: Manual hotspot start (final setup)**

* Because you want to **choose hotspot name & password every time**, we **don’t put `nmcli` in the script**.

* To start the hotspot manually after boot:

```bash
nmcli device wifi hotspot \
ifname wlp0s20f3 \
con-name pop-hotspot \
ssid "MyHotspotName" \
password "MyHotspotPass"
```

* You can change the `ssid` and `password` each time.

* Your phone connects and gets internet immediately, because NAT & forwarding are already active.

* To stop the hotspot:

```bash
nmcli connection down pop-hotspot
```

* Optional: remove NAT & forwarding rules if needed (iptables cleanup).

---

## ✅ **Summary of the final setup**

| Component                   | Purpose                                                          |
| --------------------------- | ---------------------------------------------------------------- |
| `start-hotspot.sh`          | Sets up NAT & forwarding automatically                           |
| `hotspot-network.service`   | Runs the script at boot, so rules are applied automatically      |
| `nmcli device wifi hotspot` | Manual hotspot start — lets you choose SSID & password each time |
| Optional iptables cleanup   | Keeps rules tidy if hotspot stopped                              |

**Key result:**

* Boot laptop → NAT & forwarding applied automatically
* Start hotspot manually → full control over Wi-Fi name/password
* Stop hotspot → network rules remain until reboot, optional cleanup

---

If you want, I can make a **visual workflow diagram** showing exactly **how NAT, forwarding, and manual hotspot fit together**, so it’s easy to remember.

Do you want me to do that?
