# hass
 
[https://www.home-assistant.io](Home Assistant) setup guide

## Basics

1. Flash Raspian Lite image to SD card

https://www.raspberrypi.org/downloads/raspbian/

2. Enable SSH (create empy file at root of file system called `ssh`)

`touch /Volumes/boot/ssh`

3. Add wifi network configuration
- Create file
`touch /Volumes/boot/wpa_supplicant.conf`
- Add following
```
country=AU
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1

network={
ssid="NETWORK NAME"
psk="PASSWORD"
}
```

## Booting from SSD (RPI4)

Adapted from https://www.stewright.me/2019/10/run-raspbian-from-a-usb-or-ssd-on-a-raspberry-pi-4/

1. Repeat instructions from ``Basics`` but on the USB device

2. On the rpi use fdisk to find disk name (e.g. `/dev/sda`)

```
fdisk -l
```

3. Edit `cmdline.txt` and set root to the disk name e.g.

```
root=/dev/sda rootfstype=ext4 rootwait
```

4. Reboot

5. Modify raspi-config to enable expansion of USB disk

```
# Login as root
sudo -i
# Copy raspi-config to home folder
cp /usr/bin/raspi-config ~
# Modify the copy to replace SD device name with USB name
sed -i 's/mmcblk0p/sda/' ~/raspi-config
sed -i 's/mmcblk0/sda/' ~/raspi-config
```

6. Run modified raspi-config and expand partition (Advanced Options > Expand Filesystem)

```
~/raspi-config
```

7. Reboot

8. Confirm it worked

```
df -h
```

## Install Pre-reqs

### rpi basic config

1. Configure your pi and set the following:
    - timezone
    - hostname

`sudo raspi-config`

### postgres

1. Install postgres

```
sudo apt update
sudo apt install postgresql postgresql-contrib
```

2. Check that postgres service is enabled and running

```
sudo service postgresql status
```

3. Configure postgres

```
sudo -u postgres psql
CREATE USER hass WITH PASSWORD 'raspberry';
CREATE DATABASE hass;
GRANT ALL PRIVILEGES ON DATABASE hass TO hass;
```

### MQTT

1. Install MQTT

```
sudo apt update
sudo apt install mosquitto mosquitto-clients
```

2. Check that mosquitto service is enabled and running

```
sudo service mosquitto status
```

## Install home assistant

Adatped from https://www.home-assistant.io/docs/installation/virtualenv/

1. Install py3 virtual environments

```
sudo apt-get install python3-venv
```

2. Create virtual env

```
python3 -m venv homeassistant
```

3. Activate virtual env

```
source homeassistant/bin/activate
```

4. Install hass

```
python3 -m pip install homeassistant
```

## Configure home assistant

1. Install Python postgres library

```
source homeassistant/bin/activate
pip install psycopg2
```

2. Edit hass configuration file

```
nano .homeassistant/configuration.yml
```

```
# Configure a default setup of Home Assistant (frontend, api, etc)
default_config:

# Uncomment this if you are using SSL/TLS, running in Docker container, etc.
# http:
#   base_url: example.duckdns.org:8123

history:

recorder:
  db_url: postgresql://hass:raspberry@127.0.0.1:5432/hass

# Text to speech
tts:
  - platform: google_translate

group: !include groups.yaml
automation: !include automations.yaml
script: !include scripts.yaml
scene: !include scenes.yaml

mqtt:
  broker: 127.0.0.1
  discovery: true
  discovery_prefix: homeassistant

# TP-LINK devices need manual ip addresses, otherwise the devices do not get detected after a reboot!
tplink:
  discovery: false
  switch:
    - host: 192.168.0.11
    - host: 192.168.0.12
```


## Create systemd service for home assistant

1. Create service file

```
sudo nano /etc/systemd/system/hass.service
```

2. Add service description

```
[Unit]
Description=Home Assistant
After=mosquitto.service postgresql.service

[Service]
User=pi
StandardOutput=syslog
StandardError=syslog
WorkingDirectory=/home/pi/homeassistant
ExecStart=/home/pi/homeassistant/bin/hass
Restart=always
KillSignal=SIGQUIT

[Install]
WantedBy=multi-user.target
```

3. Enable and start service

```
sudo systemctl enable hass.service
sudo service hass start
```
