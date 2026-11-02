
⚠️ DEPRECATED — This project is no longer maintained
This version of OpenBTS-UMTS is deprecated and will no longer receive updates or bug fixes.
Please use the newer version instead:
👉 https://github.com/rldv1/kapusta
The old repository remains available for archival and reference purposes.
Thank you to everyone who has used or contributed to this project!




# OpenBTS-UMTS + OsmoHLR 3G
🚀 Version 1.1 Released!

Major fixes to mobile data functionality and connection stability!

✅ Confirmed working devices: Samsung Galaxy A42 5G, Galaxy J5, and Galaxy S7.

⚠️ Note: Mobile data activation may take longer than 30 seconds. This is normal. Please wait at least 45 seconds for the data indicator to appear before troubleshooting.







«[!WARNING]
⚠️ Experimental Software: Known Data Connectivity Issues

This OpenBTS-UMTS stack is still experimental and under active development. While UMTS registration, authentication, and packet data are functional, some users may experience issues with mobile data connectivity, including successful network attachment without working internet access.

If you encounter any issues, please report them through the "GitHub Issues" (https://github.com/kingyatrib/OpenBTS-UMTS-OsmoHLR-3G/issues) page. Include your device model, relevant logs, and a description of the problem whenever possible.

Future updates and bug fixes are planned, including improvements to packet data reliability, radio functionality, and overall stability.

Thank you for testing and helping improve this project!»


**Hyper-early / experimental. Lab use only.**

Patched OpenBTS-UMTS with:

- USRP B210/B200-family support
- OsmoHLR + Milenage UMTS AKA
- native UMTS integrity keys
- successful Attach + PDP Context
- working 3G packet data through `sgsntun`

Tested on Ubuntu 24.04 with UHD 4.6.x.

> Use only on frequencies and RF setups you are legally allowed to transmit on.  
> Never upload real SIM `K`, `Ki`, `OP`, `OPc`, SQN, or HLR subscriber databases.

---

## 1. Install requirements

```bash
sudo apt update

sudo apt install -y \
  build-essential \
  gcc g++ make \
  autoconf automake libtool libtool-bin \
  pkg-config git \
  sqlite3 libsqlite3-dev \
  libusb-1.0-0-dev \
  libreadline-dev \
  libzmq3-dev \
  libosip2-dev \
  libortp-dev \
  libboost-all-dev \
  libuhd-dev uhd-host \
  osmo-hlr \
  libosmo-gsup-client-dev \
  libosmocore-dev \
  libtalloc-dev \
  tcpdump iptables \
  telnet
```

If `telnet` is unavailable on your distro, install its equivalent, for example:

```bash
sudo apt install inetutils-telnet
```

The original OpenBTS-UMTS dependency list is older, so some package names differ on modern Ubuntu. This repo was brought up on Ubuntu 24.04.

---

## 2. Clone

```bash
git clone https://github.com/kingyatrib/OpenBTS-UMTS-OsmoHLR-3G.git
cd OpenBTS-UMTS-OsmoHLR-3G
```

The patched OpenBTS source is in:

```bash
cd source
```

---

## 3. Check the USRP

Connect the B210/B200 and run:

```bash
uhd_find_devices
uhd_usrp_probe
```

Make sure UHD can see the radio before starting OpenBTS.

---

## 4. Start OsmoHLR

```bash
sudo systemctl enable --now osmo-hlr
```

Check:

```bash
systemctl status osmo-hlr --no-pager
ss -ltnp | grep -E ':(4222|4258)'
```

The working setup expects:

```text
GSUP: 127.0.0.1:4222
VTY:  127.0.0.1:4258
```

A redacted example config is included at:

```text
docs/osmo-hlr.PUBLIC.cfg
```

---

## 5. Add your lab SIM / USIM

Connect to OsmoHLR:

```bash
telnet 127.0.0.1 4258
```

Then:

```text
enable
subscriber imsi <IMSI> create
subscriber imsi <IMSI> update msisdn <MSISDN>
subscriber imsi <IMSI> update aud3g milenage k <K_HEX> opc <OPC_HEX>
```

Use only your own lab subscriber values.

---

## 6. Build the GSUP helper

From the repository root:

```bash
gcc -O2 -Wall -Wextra \
  tools/openbts-gsup-auth.c \
  -o openbts-gsup-auth \
  $(pkg-config --cflags --libs \
    libosmo-gsup-client libosmocore libosmogsm) \
  -ltalloc
```

Install it:

```bash
sudo install -m 0755 ./openbts-gsup-auth \
  /usr/local/bin/openbts-gsup-auth
```

Test:

```bash
/usr/local/bin/openbts-gsup-auth <IMSI>
```

Expected style of output:

```text
AKA RAND=... AUTN=... XRES=... CK=... IK=...
```

---

## 7. Build OpenBTS-UMTS

```bash
cd source

./autogen.sh
./configure
make -j"$(nproc)"
```

This repository already contains the patches. Do **not** apply `WORKING_TREE.patch` again.

---

## 8. Configure OpenBTS

The working configuration values are saved in:

```text
docs/CURRENT_OPENBTS_CONFIG.txt
```

Important groups:

```text
UMTS.Identity.*
UMTS.Radio.*
GPRS.NMO
GPRS.RAC
UMTS.Best.Effort.BytesPerSec
UMTS.UseTurboCodes
GGSN.*
```

Configure MCC/MNC, UARFCN/band, power, and other RF values for your own legal lab setup.

---

## 9. Start everything

### Terminal 1: OsmoHLR

```bash
sudo systemctl start osmo-hlr
```

### Terminal 2: transceiver

From `TransceiverUHD/`:

```bash
sudo ./transceiver 1
```

### Terminal 3: OpenBTS-UMTS

From `source/`:

```bash
sudo ./apps/OpenBTS-UMTS
```

Wait for:

```text
system ready
```

A good attach should eventually show:

```text
Authentication
Security Mode Complete
Attach Complete
Activate PDP Context
Radio Bearer Setup Complete
```

---

## 10. Enable Internet for the phone

First enable forwarding:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

Find your Internet interface:

```bash
WAN="$(ip route show default | awk '/default/ {print $5; exit}')"
echo "$WAN"
```

The development setup used:

```text
192.168.99.0/24
```

If your GGSN uses a different UE subnet, change it below.

```bash
UE_SUBNET="192.168.99.0/24"
```

Add NAT:

```bash
sudo iptables -t nat -A POSTROUTING \
  -s "$UE_SUBNET" -o "$WAN" -j MASQUERADE
```

Allow forwarding:

```bash
sudo iptables -A FORWARD \
  -i sgsntun -o "$WAN" -j ACCEPT

sudo iptables -A FORWARD \
  -i "$WAN" -o sgsntun \
  -m conntrack --ctstate ESTABLISHED,RELATED \
  -j ACCEPT
```

Watch phone traffic:

```bash
sudo tcpdump -ni sgsntun
```

If packets appear there, the UMTS side is working.

---

## Optional: forward a port to the UE

You do **not** need this for normal mobile Internet.

Example: forward TCP port `8080` on the Linux PC to `192.168.99.1:8080`:

```bash
WAN="$(ip route show default | awk '/default/ {print $5; exit}')"
UE_IP="192.168.99.1"

sudo iptables -t nat -A PREROUTING \
  -i "$WAN" -p tcp --dport 8080 \
  -j DNAT --to-destination "$UE_IP:8080"

sudo iptables -A FORWARD \
  -i "$WAN" -o sgsntun \
  -p tcp -d "$UE_IP" --dport 8080 \
  -j ACCEPT
```

If the Linux PC is behind a home router, that router must also forward the port to the Linux PC.

---

## Quick troubleshooting

### USRP not found

```bash
uhd_find_devices
uhd_usrp_probe
```

### OsmoHLR / GSUP not listening

```bash
systemctl status osmo-hlr --no-pager
ss -ltnp | grep 4222
```

### PDP activates but no Internet

```bash
ip link show sgsntun
sudo tcpdump -ni sgsntun
sysctl net.ipv4.ip_forward
ip route
sudo iptables -t nat -S
sudo iptables -S FORWARD
```

### See the development packet probes

```bash
grep -Rni '### ' UMTS SGSNGGSN
```

---

## Status

This is **hyper-early experimental software**.

It works in the current test setup, but expect bugs, dependency differences, RF/hardware quirks, and incomplete features.

The deeper patch/history notes are in:

```text
SETUP_GUIDE.md
docs/
```

## UMTS compatibility and build notes

### Transceiver symlink

After compiling, run from the source directory:

`cd source`

`ln -s TransceiverUHD/transceiver transceiver`

Only create the symlink if `source/transceiver` does not already exist.

### Starting the UHD transceiver

Open a separate terminal and run:

`cd ~/OpenBTS-UMTS-OsmoHLR-3G/source/TransceiverUHD`

`sudo ./transceiver 1`

Keep the transceiver running while using OpenBTS-UMTS.
Run OpenBTS-UMTS in another terminal.

### Samsung J5 packet-data fixes

- Fixed RRC Radio Bearer Setup by remapping existing signalling bearers.
- Fixed stale GMM-to-UE associations after RRC reconnection.
- Prevented downlink packets from being routed to deleted UE contexts.
- Successfully tested ### Optional: Extend UMTS Inactivity Timers

If devices experience problems resuming packet data after being idle, you can optionally increase the network's inactivity timers.

**1. Open the CLI in a separate terminal:**

```bash
cd ~/OpenBTS-UMTS-OsmoHLR-3G/source
sudo ./apps/OpenBTS-UMTSCLI
```

**2. Set the inactivity timers:**

```text
rawconfig UMTS.Timers.Inactivity.Release 86400
rawconfig UMTS.Timers.Inactivity.Delete 172800
```

This changes the following settings:

| Setting | Default | New value |
|---|---|---|
| RRC inactivity release | 180 seconds | 24 hours |
| Idle UE deletion | 300 seconds | 48 hours |

Restart OpenBTS-UMTS to ensure the new settings take effect. No recompilation is required.

**Important:** These values exceed the normal configuration limits, so `rawconfig` is required instead of `config` or `devconfig`.

This is an **experimental workaround, not a permanent fix** for idle-reconnection issues. Phones may still voluntarily release their RRC connections, and longer inactivity timers may increase resource usage and battery consumption.

To restore the original timer values:

```text
rawconfig UMTS.Timers.Inactivity.Release 180
rawconfig UMTS.Timers.Inactivity.Delete 300
```UMTS packet data with Galaxy A42 and Galaxy J5.



never forget to back up : sudo cp -a /etc/OpenBTS/OpenBTS-UMTS.db /etc/OpenBTS/OpenBTS-UMTS.db.bak


### Experimental project

This is an experimental proof of concept demonstrating Milenage AKA
and real UMTS packet data with OpenBTS-UMTS.

It is not plug-and-play. Expect debugging and compatibility issues.

For major problems, visit the DIY Discord:
https://discord.gg/CqW9XKst9E

Tag the project maintainer for assistance.
