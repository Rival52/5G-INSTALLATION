# 5G-INSTALLATION
5G Core et UE with 2 PC

# 📡 srsRAN 5G SA + Open5GS 

## Architecture

```
PC1 (srsUE + B210)
        ⇅ (câble RF + atténuateur)
PC2 (gNB srsRAN + B210 + Open5GS Docker)
```

---

## Prérequis (PC1 & PC2)

### CPU en mode performance

```bash
# Activer le mode performance
for i in /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor; do
    echo performance | sudo tee $i
done

# Vérifier
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
```

---

## 🖥️ PC2 — Installation dépendances

### 📦 UHD + GPSDO

```bash
# OCXO 
# GPDO integrated on USRP

<p align="center">
  <img src="https://github.com/SitrakaResearchAndPOC/OCXO_USRP/blob/main/gpsdo1.png" width="200" />
  <img src="https://github.com/SitrakaResearchAndPOC/OCXO_USRP/blob/main/gpsdo2.png" width="200" />
  <img src="https://github.com/SitrakaResearchAndPOC/OCXO_USRP/blob/main/gpsdo3.png" width="235" />
</p>

# Installation  
Just ignore docker manipulation for real manipulation
```
xhost +
```
```
docker image pull ubuntu:24.04
```
```
docker run -tid --privileged -v /dev/bus/usb:/dev/bus/usb -v /tmp/.X11-unix:/tmp/.X11-unix:ro -v $XAUTHORITY:/home/user/.Xauthority:ro --net=host --env="DISPLAY=$DISPLAY" --env="LC_ALL=C.UTF-8" --env="LANG=C.UTF-8" --name gpsdo_03_2026 ubuntu:24.04
```
```
docker exec -ti gpsdo_03_2026 /bin/bash
```
Directly there for non docker installation
```
apt update
```
```
apt install -y git cmake build-essential libboost-all-dev     libusb-1.0-0-dev python3-dev python3-mako python3-numpy     python3-requests python3-setuptools libfftw3-dev     libcomedi-dev libgps-dev libgmp-dev swig pkg-config gedit
```
```
git clone https://github.com/EttusResearch/uhd.git
```
```
cd uhd
```
```
ls
```
```
cd ..
```
```
gedit uhd/host/lib/usrp/gps_ctrl.cpp
```
Full code is there [gps_ctrl.cpp](https://github.com/SitrakaResearchAndPOC/OCXO_USRP/blob/main/gps_ctrl.cpp)    
For futur manipulation patch is more interesting :  <br/>

<p align="center"> <img src="https://github.com/SitrakaResearchAndPOC/OCXO_USRP/blob/main/gpsdo_patch.png" alt="description" width=550 /> </p>

* STEP1 :  <br/>
Change regex use :  <br/>
```
static const std::regex gp_msg_regex("^\\$G.*$");
```
instead  <br/>
```
static const std::regex gp_msg_regex("^\\$GP.*,\\*[0-9A-F]{2}$");
```

* STEP2 :  <br/>
Before :
```
msgs[msg.substr(1, 5)] = msg;
```
add, 
```
// change GN by GP
if(msg.substr(1,2) == "GN"){ msg.replace(1, 2, "GP");}
```
Compilation of UHD  
```    
cd uhd/host/
```
```
ls
```
```
mkdir build
```
```
cd build/
```
Verify option ENABLE_PYTHON
```
cmake .. -LAH | grep -i ENABLE_PYTHON
```
Verify option ENABLE_LIBUHD
```
cmake .. -LAH | grep -i ENABLE_LIBUHD
```
Cmake before installation
```
cmake ..   -DENABLE_PYTHON=ON  -DENABLE_LIBUHD=ON -DPYTHON_EXECUTABLE=$(which python3)
```

Choose the real number core of computer on number 10
```
make -j 10
```
```
make install
```
```
ldconfig
```
Downloading Image of USRP
```
cd /usr/local/lib/uhd/utils/
```
```
ls
```
```
uhd_images_downloader 
```
Testing synchronization GPS : 
```
cd /usr/local/lib/uhd/utils/
```
```
ls
```
```
./query_gpsdo_sensors 
```
Nruradio log ( <bold>GPS ALIGN </bold> is very important): 
<p align="center"> <img src="https://github.com/SitrakaResearchAndPOC/OCXO_USRP/blob/main/log_nuradio.png" alt="description" width=550 /> </p>

Adding script bonus on PYTHON
```
gedit test_usrp_gpsdo.py
```
```
#!/usr/bin/env python3
import sys
import os
# AVANT COMPILATION UHD ACTIVER API PYTHON, POUR VOIR OPTION
# cmake .. -LAH | grep -i ENABLE_PYTHON
# cmake .. -LAH | grep -i ENABLE_LIBUHD
# cmake ..   -DENABLE_PYTHON=ON  -DENABLE_LIBUHD=ON -DPYTHON_EXECUTABLE=$(which python3)


# POUR TROUVER LE CHEMIN ACTUEL, UTILISE CE SCRIPT :
# EN PYTHON
# import uhd
# print(uhd.__file__)
# EN BASH : 
# rm -rf /usr/lib/python3/dist-packages/uhd
# rm -rf /usr/lib/python3/site-packages/uhd


# --- 1. Supprimer les chemins UHD ---
# 
paths_to_remove = [
    "/usr/lib/python3/dist-packages/uhd",
    "/usr/lib/python3/site-packages/uhd",  # corrigé ici
]

for p in paths_to_remove:
    if p in sys.path:
        sys.path.remove(p)

# --- 2. Ajouter les chemins comme PYTHONPATH ---
# POUR TROUVER LE NOUVEAU CHEMIN (voir aprés ldconfig de compilation uhd pour nouveau chemin ) 
# cmake .. -LAH | grep -i PYTHONPATH
# EN BASH : 
# export PYTHONPATH=/usr/local/lib/python3.12/dist-packages:$PYTHONPATH
# export PYTHONPATH=/usr/local/lib/python3.12/site-packages:$PYTHONPATH

new_paths = [
    "/usr/local/lib/python3.12/dist-packages",
    "/usr/local/lib/python3.12/site-packages",
]

for p in new_paths:
    if p not in sys.path:
        sys.path.insert(0, p)  # priorité haute

# --- 3. Synchroniser avec la variable d'environnement ---
os.environ["PYTHONPATH"] = ":".join(new_paths) + ":" + os.environ.get("PYTHONPATH", "")

# --- 4. Vérification ---
print("=== Vérification des chemins ajoutés ===")
for p in new_paths:
    if p in sys.path:
        print(f"[OK] {p} est présent dans sys.path")
    else:
        print(f"[ERREUR] {p} n'est PAS présent")

print("\n=== sys.path actuel ===")
for p in sys.path:
    print(p)

import uhd
from uhd import usrp
import time

usrp_device = usrp.MultiUSRP("")

# Utiliser le GPSDO comme source d'horloge
usrp_device.set_clock_source("gpsdo")
usrp_device.set_time_source("gpsdo")
time.sleep(2)

# Liste des capteurs disponibles
sensor_names = usrp_device.get_mboard_sensor_names(0)
print("Capteurs disponibles :", sensor_names)

# Vérifie GPS lock proprement
if "gps_locked" in sensor_names:
    try:
        gps_sensor = usrp_device.get_mboard_sensor("gps_locked", 0)
        print(f"GPSDO lock sur satellites : {gps_sensor.value}")
    except RuntimeError as e:
        print("GPS non verrouillé (pas de fix pour le moment)")
        print("Détail :", e)
else:
    print("Capteur 'gps_locked' non disponible")

# Vérifie ref_locked
if "ref_locked" in sensor_names:
    try:
        ref_sensor = usrp_device.get_mboard_sensor("ref_locked", 0)
        print(f"ref_locked = {ref_sensor.value}")
    except RuntimeError as e:
        print("Impossible de lire ref_locked :", e)

# ------------------------------------------------------------------
# 🔥 ALIGNEMENT TEMPOREL GPSDO
# ------------------------------------------------------------------
print("\n--- Synchronisation temporelle GPSDO ---")

time.sleep(2)

usrp_device.set_time_now(uhd.types.TimeSpec(0.0))
usrp_device.set_time_next_pps(uhd.types.TimeSpec(0.0))

print("Attente du PPS GPSDO...")
time.sleep(2)

# ------------------------------------------------------------------
# 🔥 INFO GPS / PPS (STYLE UHD PROPRE)
# ------------------------------------------------------------------

print("\n--- Infos GPS / PPS ---")

try:
    gps_time = float(usrp_device.get_mboard_sensor("gps_time", 0).value)

    last_pps = int(gps_time)

    print(f"last_pps: {last_pps} vs gps: {last_pps}")

except Exception as e:
    print("Impossible de lire gps_time :", e)

print("\nPrinting available NMEA strings:")

try:
    print("GPS_GPGGA:", usrp_device.get_mboard_sensor("gps_gpgga", 0).value)
    print("GPS_GPRMC:", usrp_device.get_mboard_sensor("gps_gprmc", 0).value)
except Exception as e:
    print("NMEA non disponible :", e)

try:
    gps_time = float(usrp_device.get_mboard_sensor("gps_time", 0).value)
    print(f"GPS Epoch time at last PPS: {gps_time:.5f} seconds")
except Exception as e:
    print("GPS epoch non disponible :", e)

# ------------------------------------------------------------------
# 🔥 UHD DEVICE TIME (CORRIGÉ STYLE QUERY_SENSORS_GPSDO)
# ------------------------------------------------------------------

gps_time = float(usrp_device.get_mboard_sensor("gps_time", 0).value)

device_time = usrp_device.get_time_now().get_real_secs()

# reconstruction propre comme UHD tool (cohérent, pas FPGA brut)
absolute_time = gps_time + (device_time - int(device_time))

print(f"UHD Device time last PPS:   {gps_time:.5f} seconds")
print(f"UHD Device time right now:  {absolute_time:.5f} seconds")

print(f"PC Clock time:              {int(time.time())} seconds")

print("\nDone!")
```
```
python3 test_usrp_gpsdo.py
```
```

### 🐳 Docker

```bash
sudo apt-get install -y docker.io docker-compose-plugin
sudo usermod -aG docker $USER
newgrp docker
```

---

## 🏗️ Installation gNB (PC2)

```bash
# Cloner
git clone https://github.com/srsran/srsRAN_Project.git
cd srsRAN_Project

# Build
mkdir build && cd build
cmake .. \
  -DUHD_FOUND=ON \
  -DUHD_LIBRARIES=/usr/local/lib/libuhd.so \
  -DUHD_INCLUDE_DIRS=/usr/local/include \
  -DENABLE_UHD=ON

make -j$(nproc) gnb
```

---

## 📡 Installation UE (PC1)

```bash
# Cloner
git clone https://github.com/srsran/srsRAN_4G.git
cd srsRAN_4G

# Build
mkdir build && cd build
cmake ..
make -j$(nproc) srsue
```

---

## 🌐 Configuration Open5GS (PC2)

### 🔎 1. Trouver IP PC2

```bash
ip addr show | grep "inet " | grep -v "127\|10.45\|10.53\|172"
```

---

### ⚙️ 2. Modifier open5gs.env

```bash
nano ~/srsRAN_Project/docker/open5gs/open5gs.env
```

```env
MONGODB_IP=127.0.0.1
OPEN5GS_IP=<IP_PC2>
UE_IP_BASE=10.45.0
UPF_ADVERTISE_IP=127.0.0.7
DEBUG=false
SUBSCRIBER_DB=subscriber_db.csv
NETWORK_NAME_FULL=srsRAN
NETWORK_NAME_SHORT=srsRAN
TZ=Indian/Antananarivo
```

---

### 👤 3. Créer abonnés

```bash
cat << 'EOF' > ~/srsRAN_Project/docker/open5gs/subscriber_db.csv
name,imsi,key,op_type,op_c,amf,qci,ip_alloc
ue1,001010123456780,00112233445566778899aabbccddeeff,opc,63bfa50ee6523365ff14c1f45f88737d,8000,9,10.45.1.2
ue2,001010123456781,00112233445566778899aabbccddeeff,opc,63bfa50ee6523365ff14c1f45f88737d,8000,9,10.45.1.3
EOF
```

---

### 🛠️ 4. Patch setup_tun.py

```bash
nano ~/srsRAN_Project/docker/open5gs/setup_tun.py
```

➡️ Remplacer complètement par :

```python
# (coller ici ton script complet corrigé)
```

---

### 🔧 5. Vérifier open5gs-5gc.yml

```bash
grep -n "^upf:" ~/srsRAN_Project/docker/open5gs/open5gs-5gc.yml
grep -n "ngap" ~/srsRAN_Project/docker/open5gs/open5gs-5gc.yml
```

✔️ UPF doit contenir :

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

### 🐳 6. Docker compose

```bash
nano ~/srsRAN_Project/docker/docker-compose.yml
```

✔️ Vérifier :

```yaml
services:
  5gc:
    container_name: open5gs_5gc
    privileged: true
    network_mode: host
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun:/dev/net/tun
```

---

## 📶 Configuration gNB (PC2)

```bash
nano ~/gnb_uhd.yaml
```

⚠️ Remplacer `<IP_PC2>`

---

## 📱 Configuration UE (PC1)

```bash
nano ~/ue_uhd.conf
```

⚠️ Important :

```ini
[gw]
ip_netmask = 255.255.0.0
```

---

## 🚀 Script démarrage 5G (PC2)

```bash
cat << 'EOF' > ~/start_5g.sh
#!/bin/bash
echo "=== Démarrage 5G ==="

for i in /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor; do
    echo performance | sudo tee $i > /dev/null
done

sudo ip link delete ogstun 2>/dev/null || true

cd ~/srsRAN_Project/docker
docker compose down 2>/dev/null
docker compose up -d 5gc
sleep 15

sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -F
sudo iptables -t nat -A POSTROUTING -s 10.45.0.0/16 ! -o ogstun -j MASQUERADE
sudo iptables -I FORWARD 1 -j ACCEPT

echo "✅ 5GC prêt"
EOF

chmod +x ~/start_5g.sh
```

---

## 🔗 Script UE (PC1)

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

## ▶️ Démarrage

### PC2

```bash
sudo ~/start_5g.sh

cd ~/srsRAN_Project/build/apps/gnb
sudo taskset -c 6-11 ./gnb -c ~/gnb_uhd.yaml
```

---

### PC1

```bash
# Terminal 1
sudo taskset -c 0-11 ./srsue ~/ue_uhd.conf

# Terminal 2
~/fix_ue_route.sh
```

---

## 🔁 Mise à jour IP

```bash
NEW_IP=$(ip addr show | grep "inet " | grep -v "127\|10.45\|10.53\|172\|docker" | awk '{print $2}' | cut -d'/' -f1 | head -1)

sed -i "s/OPEN5GS_IP=.*/OPEN5GS_IP=$NEW_IP/" ~/srsRAN_Project/docker/open5gs/open5gs.env
sed -i "s/addr: .*/addr: $NEW_IP/" ~/gnb_uhd.yaml
sed -i "s/bind_addr: .*/bind_addr: $NEW_IP/" ~/gnb_uhd.yaml
```

---

## 🔍 Vérifications

```bash
ip addr show ogstun
docker logs open5gs_5gc | grep PFCP
sudo ss -ulnp | grep 2152
docker logs open5gs_5gc | grep gNB
docker logs open5gs_5gc | grep UPF
```

---

## ⚠️ Troubleshooting

| Problème           | Solution                |
| ------------------ | ----------------------- |
| UE ne connecte pas | `sudo ip netns add ue1` |
| Pas d’internet     | vérifier NAT            |
| AMF refusé         | IP incorrecte           |
| Packet loss        | iptables                |

---

## ✅ Résultat attendu

* UE attaché ✅
* PDU session OK ✅
* Ping Internet OK ✅

---
