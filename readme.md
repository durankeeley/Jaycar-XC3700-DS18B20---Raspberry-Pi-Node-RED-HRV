# Jaycar XC3700 (DS18B20) - Raspberry Pi Node-RED HRV

## Overview

Creating a home ventilation system (like HRV) using a Raspberry Pi, Node-RED and MQTT to control a Tasmota Wi-Fi plug.

## Wiring Diagram for the DS18B20

![pi-to-hrv_bb](Jaycar DS18B20 - Raspberry Pi (Node-RED HRV).assets/pi-to-hrv_bb.svg)

*Breadboard is not needed*

GND is connected to GND (Pin 6), 5V is connected to 5V (Pin 2) and OneWire data is connected to GPIO04 (Pin 7)

## Prerequisites

Enable OneWire from raspi-config

Install MQTT as a broker if are going to use the Raspberry Pi as the broker

Install Node-RED

## Sensor Code

Get the address of your OneWire Sensor by using ls on the directory:

```bash
ls /sys/bus/w1/devices/
```

EG mine is "28-011441c00aaa" so get rid of the "28-" to give me the address of "011441c00aaa"

### Sensor

Create a python file with your sensor info EG:

```bash
nano ~/sensor-[ONEWIRE_ADDRESS].py
```

Uncomment auth if you are using MQTT Authentication, fill in [USERNAME], [PASSWORD] and [ONEWIRE_ADDRESS]

 *NOTE: Remove the square brackets [ ]*

```python
#!/usr/bin/python3

import glob
import time
import paho.mqtt.publish as publish

Broker = '127.0.0.1'
#auth = {
#    'username': '[USERNAME]',
#    'password': '[PASSWORD]',
#}

pub_topic = 'temperature/sensor-[ONEWIRE_ADDRESS]' # Or whereever you would like to publish

base_dir = '/sys/bus/w1/devices/'
device_folder = glob.glob(base_dir + '28-[ONEWIRE_ADDRESS]')[0]
device_file = device_folder + '/w1_slave'

def read_temp():
    valid = False
    temp = 0
    with open(device_file, 'r') as f:
        for line in f:
            if line.strip()[-3:] == 'YES':
                valid = True
            temp_pos = line.find(' t=')
            if temp_pos != -1:
                temp = float(line[temp_pos + 3:]) / 1000.0

    if valid:
        return temp
    else:
        return None


while True:
    temp = read_temp()
    if temp is not None:
        publish.single(pub_topic, str(temp),
                hostname=Broker, port=1883,)
    time.sleep(900)

```

### Systemd

```bash
sudo nano /etc/systemd/system/mqtt-sensor-[ONEWIRE_ADDRESS].service
```

```
[Unit]
Description=MQTT Temperature Sensor [ONEWIRE_ADDRESS]
After=network.target

[Service]
ExecStartPre=/bin/sleep 30
ExecStart=/usr/bin/python3 garageDoor-sensor-mqtt.py
WorkingDirectory=/home/pi
StandardOutput=inherit
StandardError=inherit
Restart=always
User=pi

[Install]
WantedBy=multi-user.target
```

*Note: I like to add a 5s to each sensor on the sleep*

## Node-RED Flow

### Overview

Flow:

![image-20211223223125009](Jaycar DS18B20 - Raspberry Pi (Node-RED HRV).assets/image-20211223223125009.png)

Dashboard

![image-20211223223207236](Jaycar DS18B20 - Raspberry Pi (Node-RED HRV).assets/image-20211223223207236.png)

### Prerequisites

These Palette's were used to create the flow

- node-red-contrib-tasmota
- node-red-dashboard
- node-red-node-openweathermap

### Function logic for Override

```javascript
if (flow.get("hrv_override") == 'undefined') {
    flow.set("hrv_override", "hrv_automatic");
}

if (msg.payload == "hrv_automatic") {
    flow.set("hrv_override", "hrv_automatic");
}

if (msg.payload == "hrv_always_on") {
    flow.set("hrv_override", "hrv_always_on");
}

if (msg.payload == "hrv_always_off") {
    flow.set("hrv_override", "hrv_always_off");
}

if (msg.payload[0].topic === 'Inside Temperature') {
    var inside_temp_topic = msg.payload[0].topic;
    var inside_temp = Number(msg.payload[0].payload);
    var roof_temp_topic = msg.payload[1].topic;
    var roof_temp = Number(msg.payload[1].payload);
} else if (msg.payload[0].topic === 'Roof Temperature') {
    var roof_temp_topic = msg.payload[0].topic;
    var roof_temp = Number(msg.payload[0].payload);
    var inside_temp_topic = msg.payload[1].topic;
    var inside_temp = Number(msg.payload[1].payload);
}

if (flow.get("hrv_override") == "hrv_automatic") {
    if (roof_temp >= inside_temp) {
        msg.topic = "Turn on HRV";
        msg.payload = true;
        return msg;
    } else if (roof_temp < inside_temp) {
        msg.topic = "Turn off HRV";
        msg.payload = false;
        return msg;
    }
}

if (flow.get("hrv_override") == "hrv_always_on") {
        msg.topic = "HRV Override Always ON";
        msg.payload = true;
        return msg;
} 

if (flow.get("hrv_override") == "hrv_always_off") {
        msg.topic = "HRV Override Always OFF";
        msg.payload = false;
        return msg;
}
```

### Flow Export

Flow is saved out to a json file "hrv-flow.json" to import into Node-RED
