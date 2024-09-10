# Device Setup Guide
**1. Prepare the Device**

***Flash the Firmware:***

Use the firmware builder tool available at [Silicon Labs Firmware Builder](https://darkxst.github.io/silabs-firmware-builder/).

- Device: ZBDongle-E / Sonoff Zigbee 3.0 USB Dongle Plus V2 - No hardware flow control
- Firmware Version: ncp-uart-hw-v7.4.1.0-zbdonglee-230400.gbl
- Firmware Files: [Firmware Builds](https://github.com/darkxst/silabs-firmware-builder/tree/main/firmware_builds/zbdonglee)


***Verify Device Path:***

Check the device path on your system with the following command:
```
ugitho@tc-m720q:~/installation/hass$ ls -l /dev/serial/by-id/
total 0
lrwxrwxrwx 1 root root 13 Sep  8 00:08 usb-ITead_Sonoff_Zigbee_3.0_USB_Dongle_Plus_5a1d8c8c5ffded118e4bfe64024206e6-if00-port0 -> ../../ttyUSB0
```

**2. Docker Setup**

***Directory Structure:***
```
ugitho@tc-m720q:~/installation/hass$ tree -L 3
.
├── docker-compose.yml
├── home-assistant
│   └── config
├── mosquitto
│   ├── config
│   │   └── mosquitto.conf
└── zigbee2mqtt
    ├── configuration.yaml
```

****docker-compose.yml****
```
services:
  homeassistant:
    container_name: homeassistant
    image: "ghcr.io/home-assistant/home-assistant:stable"
    volumes:
      # Mount Home Assistant configuration
      - ./home-assistant/config:/config
      # Synchronize time with host
      - /etc/localtime:/etc/localtime:ro
      # Mount DBus socket for Home Assistant
      - /run/dbus:/run/dbus:ro
    restart: unless-stopped
    privileged: true
    network_mode: host

  mosquitto:
    image: eclipse-mosquitto
    container_name: mqtt
    ports:
      # Expose MQTT port
      - 1883:1883
    volumes:
      # Mount Mosquitto configuration
      - ./mosquitto/config:/mosquitto/config
      # Mount Mosquitto data
      - ./mosquitto/data:/mosquitto/data
      # Use system log for Mosquitto logs
      - mosquitto_log:/mosquitto/log

  zigbee2mqtt:
    image: koenkk/zigbee2mqtt
    container_name: zigbee2mqtt
    depends_on:
      # Ensure Mosquitto starts before Zigbee2MQTT
      - mosquitto
    volumes:
      # Mount Zigbee2MQTT data
      - ./zigbee2mqtt:/app/data
      # Mount udev for device access
      - /run/udev:/run/udev:ro
      # Synchronize time with host
      - /etc/localtime:/etc/localtime:ro
    ports:
      # Expose Zigbee2MQTT web frontend
      - 8080:8080
    devices:
      # Map Zigbee USB dongle to container
      - /dev/serial/by-id/usb-ITead_Sonoff_Zigbee_3.0_USB_Dongle_Plus_5a1d8c8c5ffded118e4bfe64024206e6-if00-port0:/dev/ttyUSB0
    restart: unless-stopped

volumes:
  # Persistent log storage for Mosquitto
  mosquitto_log:
```


****mosquitto.conf****
```
persistence true
persistence_location /mosquitto/data/
log_dest file /mosquitto/log/mosquitto.log
log_type all
listener 1883
allow_anonymous true
```

****configuration.yaml****

```
homeassistant: true
permit_join: true
mqtt:
  base_topic: zigbee2mqtt
  server: mqtt://mosquitto
serial:
  adapter: ember
  port: /dev/ttyUSB0
  baudrate: 230400
frontend:
  port: 8080
advanced:
  log_level: warn
```

**3. Launch Containers**

You can now start the containers with the following command:

```
sudo docker compose up -d --remove-orphans
```
If everything is set up correctly, there should be no errors, and you can access the web interfaces:

```
- Home Assistant: http://<your-ip-address>:8123
- Zigbee2MQTT: http://<your-ip-address>:8080
```


```
sudo docker compose down  --remove-orphans
```
**4. Home Assistant**

To integrate MQTT with Home Assistant:

1. **Open Home Assistant:** Go to your dashboard.
2. **Navigate to Integrations:**
  - Select **Settings > Devices & Services.**
  - Click **Add Integration** and search for **MQTT.**
  - Enter your MQTT broker details (IP (```<your-ip-address>```), port (```1883```), and [credentials](https://github.com/vvatelot/mosquitto-docker-compose/blob/main/README.md) if required).

After integration, Home Assistant will connect to your MQTT broker, allowing you to manage devices and sensors.

**5. Rootless container**

***Verify Device Permissions on Host***

Check the permissions and group ownership of the device, ensure the output shows:
```
ugitho@tc-m720q:~/installation/hass$ ll -l /dev/ttyUSB0
crw-rw---- 1 root dialout 188, 0 Sep  8 00:08 /dev/ttyUSB0
```

***Verify Group Membership***

Ensure your user is in the dialout group:
```
ugitho@tc-m720q:~/installation/hass$ id $USER
uid=1000(ugitho) gid=1000(ugitho) groups=1000(ugitho),4(adm),20(dialout),24(cdrom),27(sudo),30(dip),46(plugdev),100(users),114(lpadmin),125(libvirt)
```

If the dialout group is missing, add your user to the group:
```
ugitho@tc-m720q:~/installation/hass$ sudo usermod -aG dialout $USER
```

***Update Docker Compose File***

```
services:
  # ...
  zigbee2mqtt:
    user: "1000:1000"
    group_add:
      - dialout
  # ...
```

And restart the containers
