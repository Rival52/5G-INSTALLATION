# 📡 5G-INSTALLATION — srsRAN 5G SA + Open5GS + GPSDO (FULL GUIDE)

---

## 🧠 Architecture

```
PC1 (srsUE + B210)
        ⇅ (RF câble + atténuateur)
PC2 (srsRAN gNB + B210 + Open5GS Docker)
```

---

# ⚙️ PRÉREQUIS (PC1 & PC2)

## 🔧 CPU en mode performance (OBLIGATOIRE)

```bash
# Activer
for i in /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor; do
    echo performance | sudo tee $i
done

# Vérifier
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
```

---

# 🖥️ PC2 — INSTALLATION UHD

```bash
sudo apt-get install -y libuhd-dev uhd-host
sudo uhd_images_downloader
```

---

# 🐳 PC2 — INSTALLATION DOCKER

```bash
sudo apt-get install -y docker.io docker-compose-plugin
sudo usermod -aG docker $USER
newgrp docker
```

---

# 🏗️ INSTALLATION srsRAN gNB (PC2)

```bash
git clone https://github.com/srsran/srsRAN_Project.git
cd srsRAN_Project

mkdir build && cd build

cmake .. \
  -DUHD_FOUND=ON \
  -DUHD_LIBRARIES=/usr/local/lib/libuhd.so \
  -DUHD_INCLUDE_DIRS=/usr/local/include \
  -DENABLE_UHD=ON

make -j$(nproc) gnb
```

---

# 📡 INSTALLATION srsUE (PC1)

```bash
git clone https://github.com/srsran/srsRAN_4G.git
cd srsRAN_4G

mkdir build && cd build

cmake ..
make -j$(nproc) srsue
```

---

# 🌐 CONFIGURATION OPEN5GS (PC2)

---

## 1️⃣ Trouver IP PC2

```bash
ip addr show | grep "inet " | grep -v "127\|10.45\|10.53\|172"
```

➡️ Exemple : `192.168.1.102`

---

## 2️⃣ Modifier open5gs.env

```bash
nano ~/srsRAN_Project/docker/open5gs/open5gs.env
```

```env
MONGODB_IP=127.0.0.1
OPEN5GS_IP=<IP_PC2>          # ⚠️ changer si IP change
UE_IP_BASE=10.45.0
UPF_ADVERTISE_IP=127.0.0.7   # ⚠️ NE JAMAIS CHANGER
DEBUG=false
SUBSCRIBER_DB=subscriber_db.csv
NETWORK_NAME_FULL=srsRAN
NETWORK_NAME_SHORT=srsRAN
TZ=Indian/Antananarivo
```

---

## 3️⃣ Créer subscriber_db.csv

```bash
cat << 'EOF' > ~/srsRAN_Project/docker/open5gs/subscriber_db.csv
name,imsi,key,op_type,op_c,amf,qci,ip_alloc
ue1,001010123456780,00112233445566778899aabbccddeeff,opc,63bfa50ee6523365ff14c1f45f88737d,8000,9,10.45.1.2
ue2,001010123456781,00112233445566778899aabbccddeeff,opc,63bfa50ee6523365ff14c1f45f88737d,8000,9,10.45.1.3
EOF
```

---

## 4️⃣ PATCH setup_tun.py (BUG 256 IPs)

```bash
cat << 'EOF' > ~/srsRAN_Project/docker/open5gs/setup_tun.py
#!/usr/bin/env python3
import click
import ipaddress
import iptc
from pyroute2 import IPRoute
from pyroute2.netlink import NetlinkError

def handle_ip_string(ctx, param, value):
    try:
        ret = ipaddress.ip_network(value)
        return ret
    except ValueError:
        raise click.BadParameter(f'{value} is not a valid IP range.')

def iptables_add_masquerade(if_name, ip_range):
    chain = iptc.Chain(iptc.Table(iptc.Table.NAT), "POSTROUTING")
    rule = iptc.Rule()
    rule.src = ip_range
    rule.out_interface = if_name
    target = iptc.Target(rule, "MASQUERADE")
    rule.target = target
    chain.insert_rule(rule)

def iptables_allow_all(if_name):
    chain = iptc.Chain(iptc.Table(iptc.Table.FILTER), "INPUT")
    rule = iptc.Rule()
    rule.in_interface = if_name
    target = iptc.Target(rule, "ACCEPT")
    rule.target = target
    chain.insert_rule(rule)

@click.command()
@click.option("--if_name", default="ogstun", help="TUN interface name.")
@click.option("--ip_range", default='10.45.0.0/24', callback=handle_ip_string,
              help="IP range of the TUN interface.")
def main(if_name, ip_range):
    first_ip_addr = next(ip_range.hosts(), None)
    if not first_ip_addr:
        raise ValueError('Invalid IP range.')
    first_ip_addr = first_ip_addr.exploded
    ip_netmask = ip_range.prefixlen
    ipr = IPRoute()
    try:
        ipr.link('add', ifname=if_name, kind='tuntap', mode='tun')
    except NetlinkError:
        pass
    dev = ipr.link_lookup(ifname=if_name)[0]
    ipr.link('set', index=dev, state='down')
    try:
        ipr.addr('add', index=dev, address=first_ip_addr, mask=ip_netmask)
    except NetlinkError:
        pass
    ipr.link('set', index=dev, state='up')
    try:
        ipr.route('add', dst=ip_range.with_prefixlen, gateway=first_ip_addr)
    except NetlinkError:
        pass
    iptables_add_masquerade(if_name, ip_range.with_prefixlen)
    iptables_allow_all(if_name)

if __name__ == "__main__":
    main()
EOF
```

---

## 5️⃣ Vérifier open5gs-5gc.yml

```bash

```

✔️ UPF :

```yaml
upf:
  pfcp:
    server:
      - address: 127.0.0.7
  gtpu:
    server:
      - address: 127.0.0.7
        advertise: ${UPF_ADVERTISE_IP}
```

✔️ AMF :

```yaml
amf:
  ngap:
    server:
      - address: ${OPEN5GS_IP}
```

---

## 6️⃣ docker-compose.yml

```yaml
services:
  5gc:
    container_name: open5gs_5gc
    build:
      context: open5gs
      target: open5gs
      args:
        OS_VERSION: "22.04"
        OPEN5GS_VERSION: "v2.7.0"
    env_file:
      - ${OPEN_5GS_ENV_FILE:-open5gs/open5gs.env}
    privileged: true
    network_mode: host
    dns:
      - 8.8.8.8
      - 1.1.1.1
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun:/dev/net/tun
    command: 5gc -c open5gs-5gc.yml
    healthcheck:
      test: [ "CMD-SHELL", "nc -z 127.0.0.20 7777" ]
      interval: 3s
      timeout: 1s
      retries: 60
```

---

# 📶 CONFIGURATION gNB (PC2)

```bash
nano ~/gnb_uhd.yaml
```

```yaml
cu_cp:
  amf:
    addr: <IP_PC2>              # ← Changer à chaque changement de réseau
    port: 38412
    bind_addr: <IP_PC2>         # ← Changer à chaque changement de réseau
    supported_tracking_areas:
      - tac: 7
        plmn_list:
          - plmn: "00101"
            tai_slice_support_list:
              - sst: 1
  inactivity_timer: 7200

cu_up:
  ngu:
    socket:
      - bind_addr: <IP_PC2>     # ← Changer à chaque changement de réseau

ru_sdr:
  device_driver: uhd
  device_args: type=b200,clock=gpsdo
  srate: 23.04
  tx_gain: 75
  rx_gain: 75

cell_cfg:
  dl_arfcn: 368500
  band: 3
  channel_bandwidth_MHz: 20
  common_scs: 15
  plmn: "00101"
  tac: 7
  pdcch:
    common:
      ss0_index: 0
      coreset0_index: 12
    dedicated:
      ss2_type: common
      dci_format_0_1_and_1_1: false
  prach:
    prach_config_index: 1

log:
  filename: /tmp/gnb.log
  all_level: info

pcap:
  mac_enable: false
  ngap_enable: true
  ngap_filename: /tmp/gnb_ngap.pcap
```

---

# 📱 CONFIGURATION UE (PC1)

```bash
nano ~/ue_uhd.conf
```

```ini
[gw]
netns = ue1
ip_devname = tun_srsue
ip_netmask = 255.255.0.0
```

---

# 🚀 SCRIPT DÉMARRAGE PC2

```bash
cat << 'EOF' > ~/start_5g.sh
#!/bin/bash

for i in /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor; do
    echo performance | sudo tee $i > /dev/null
done

sudo ip link delete ogstun 2>/dev/null || true

cd ~/srsRAN_Project/docker
docker compose down
docker compose up -d 5gc
sleep 15

sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -F
sudo iptables -t nat -A POSTROUTING -s 10.45.0.0/16 ! -o ogstun -j MASQUERADE
sudo iptables -I FORWARD 1 -j ACCEPT

echo "5GC READY"
EOF

chmod +x ~/start_5g.sh
```

---

# 🔗 SCRIPT UE PC1

```bash
cat << 'EOF' > ~/fix_ue_route.sh
#!/bin/bash

sudo ip netns add ue1 2>/dev/null || true

while true; do
    while ! sudo ip netns exec ue1 ip link show tun_srsue > /dev/null 2>&1; do
        sleep 0.1
    done

    sudo ip netns exec ue1 ip route add default via 10.45.0.1 dev tun_srsue 2>/dev/null || true
    sudo ip netns exec ue1 ping -c10 8.8.8.8
    sleep 2
done
EOF

chmod +x ~/fix_ue_route.sh
```

---

# ▶️ DÉMARRAGE COMPLET

## PC2

```bash
sudo ~/start_5g.sh

cd ~/srsRAN_Project/build/apps/gnb
sudo taskset -c 6-11 ./gnb -c ~/gnb_uhd.yaml
```

---

## PC1

```bash
# Terminal 1
sudo taskset -c 0-11 ./srsue ~/ue_uhd.conf

# Terminal 2
~/fix_ue_route.sh
```

---

# 🔁 UPDATE IP AUTOMATIQUE

```bash
NEW_IP=$(ip addr show | grep "inet " | grep -v "127\|10.45\|10.53\|172\|docker" | awk '{print $2}' | cut -d'/' -f1 | head -1)

sed -i "s/OPEN5GS_IP=.*/OPEN5GS_IP=$NEW_IP/" ~/srsRAN_Project/docker/open5gs/open5gs.env
sed -i "s/addr: .*/addr: $NEW_IP/" ~/gnb_uhd.yaml
sed -i "s/bind_addr: .*/bind_addr: $NEW_IP/" ~/gnb_uhd.yaml
```

---

# 🔍 VÉRIFICATIONS

```bash
ip addr show ogstun
docker logs open5gs_5gc | grep PFCP
sudo ss -ulnp | grep 2152
docker logs open5gs_5gc | grep gNB
docker logs open5gs_5gc | grep UPF
```

---

# ⚠️ TROUBLESHOOTING

| Problème    | Cause            | Solution         |
| ----------- | ---------------- | ---------------- |
| Failed GW   | namespace absent | ip netns add ue1 |
| ogstun DOWN | UPF KO           | vérifier docker  |
| PDU fail    | mauvais IP UPF   | 127.0.0.7        |
| AMF refused | IP changée       | MAJ configs      |
| packet loss | NAT absent       | iptables         |
| underflows  | CPU lent         | performance      |
| bug 256 IP  | srsRAN           | patch            |

---

# ✅ RÉSULTAT FINAL

* UE attaché ✅
* Session PDU OK ✅
* Internet OK ✅
* GPSDO sync OK ✅

---
