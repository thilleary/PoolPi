# Raspberry Pi Pool Monitor
# TH Start April 14, 2026
Borrowed heavily from:
* https://github.com/fezfox/pool
* https://fezfox.com/raspberry-pi-pool-temperature-monitor/

Which borrowed heavily from:
* https://bitbucket.org/MattHawkinsUK/rpispy-pool-monitor/overview
* https://www.raspberrypi-spy.co.uk/2017/07/pool-temperature-monitoring-and-pump-control-with-the-pi-zero-w/

I wanted to monitor the temperature of my pool and back deck using a Raspberry Pi and Home Assistant.

After looking at a varity of code samples, I decided to use the repo above as a starting point.

It included everything that I needed - a simple python framework, great setup guide, MQTT integration, and a local website.

## Hardware

### Raspberry Pi Zero WH

I'm using a [Raspberry Pi Zero WH](https://www.buyapi.ca/product/raspberry-pi-zero-wireless-wh-pre-soldered-header/) which includes headers - making it easier to connect everything together - no soldering required.

You will also need a [power adapter](https://www.buyapi.ca/product/wall-adapter-power-supply-5-25v-dc-2-4a-usb-micro-b/) and MicroSD card.

![Raspberry Pi Zero WH](https://www.dropbox.com/s/lhmufa7vozinh3h/IMG_20190602_201854.jpg?raw=1)

### DS18B20 Temperature Sensors

I'm using two [DS18B20](https://www.amazon.ca/gp/product/B012C597T0/ref=ppx_yo_dt_b_asin_title_o01_s00?ie=UTF8&psc=1) to monitor the temperature - one in the pool, the other on my back deck.

![DS18B20](https://www.dropbox.com/s/qu1bncirjp3p7ye/DS18B20.jpg?raw=1)

### 4.7k Resistor

You will need a [4.7k resistor](https://www.buyapi.ca/product/resistor-4-7k-ohm-14w-5-axial-pack-of-10/).

### Wires

I purchased a large set of [Jumper Wires](https://www.amazon.ca/gp/product/B01C84WKN0/ref=ppx_yo_dt_b_asin_title_o01_s00?ie=UTF8&psc=1) for other projects.

You will also need wires to run from each DS18B20 sensor to the Pi. I'm using regular speaker wire.

## GPIO PINs

I recommend [this site](https://pinout.xyz) for good diagrams of the Pi GPIO pins.

### Wiring

* Pi GPIO Pin 1 (+3.3v) to each DS18B20 (red)
* Pi GPIO Pin 7 (Data) to each DS18B20 (yellow)
* Pi GPIO Pin 9 (GND) to each DS18B20 (black)
* 4.7k resistor between 3.3v and data.

![Wiring](https://www.dropbox.com/s/idwx8tkvhhqfvsd/Wiring.png?raw=1)

## Software Setup

### Setup SSH
The easiest way to use the pi is via SSH. Im assuming you've already got an SD card with Raspbian on it. Put that card into your computer however you do it and using a simple text editor create a blank file called `ssh` in the root. Then create another file called `wpa_supplicant.conf` with the following data:

```
country=us
update_config=1
ctrl_interface=/var/run/wpa_supplicant

network={
 scan_ssid=1
 ssid="YOUR_NETWORK_SSID"
 psk="YOUR_NETWORK_PASSWORD"
}
```

This means when you boot up your pi it will connect to your wifi, and you can ssh to it. You just need to know your pis IP address, and then you can call:
`ssh pi@YOUR_PI_IP_ADDRESS` and the default password is `raspberry`. Logging in will take you to the default non-root user "pi". This account's path is  `/home/pi/`.

### Install packages
Once logged into the pi, update:
```
sudo apt-get update
sudo apt-get -y upgrade
```
That will take 10-20 minutes...then install these packages:
```
sudo apt-get -y install git
sudo apt-get install python3-gpiozero
sudo apt-get -y install python3-pip
sudo pip3 install flask
sudo pip3 install requests
sudo pip3 install paho-mqtt
sudo apt-get install mosquitto
sudo apt-get install mosquitto-clients
```

### Clone the repo
My repo contains all the code, described below, to monitor the temperature from two sensors. Clone the git repo:
```
git clone https://github.com/shawn-peterson/PoolPi.git
```

Navigate to it:
```
cd pool
```

Make the launcher executable:
```
chmod +x launcher.sh
```

### Setup cron
This will make the launcher code run everytime the pi reboots:
```
sudo crontab -e
```
Add this line to the bottom:
```
@reboot sh /home/pi/pool/launcher.sh > /home/pi/pool/logs/cronlog 2>&1
```

### Reboot the pi
Rebooting the pi will start the web page and logger:
```
sudo reboot now
```

## Software
Below are the two ways I am using this software:

1. View/control from a local website using flask
1. Publish to a mosquitto channel, which can be monitored using [Home Assistant](https://www.home-assistant.io)

### Post to a webpage on your local network using flask
After reboot, launcher will run `webmain.py` which will use flask to create a web page on your local website. This can be accessed by navigating to you pi's IP address followed by port 5000:
```
YOUR_PI_IP_ADDRESS:5000
```
You will need to login with the username "admin" and the password "splishsplosh". You will see something like this:

![Website](https://www.dropbox.com/s/jzusskag3eoicao/Web.JPG?raw=1)

If not check your logs. If you have any familiarity with writing webpages, you can edit the files found in "templates".

The index page will display the most current temperature from each sensor.

### Publish and Subscribe to a mosquitto channel
The code in `poolmain.py` creates and publishes to a *MQTT broker*. This is the setup here.

As Home Assistant only supports one MQTT at a time, I've added the ability to connect to a remote MQTT - otherwise it will use the local Pi.

```
mqtt_ip = p.getIp()

#Use a remote MQTT server when specified
if c.MQTTIP:
    mqtt_ip = c.MQTTIP

#Setup MQTT broker details
client = mqtt.Client("PoolMain")
client.username_pw_set("mqtt", c.MQTTPWORD)
client.connect(mqtt_ip, 1883, 60)
client.loop_start()
```

You can test this by logging into the pi with another terminal window, and *subscribing* to this channel:
```
mosquitto_sub -v -t "pool/temperature/#"
```

Every 5 minutes you will see this in your terminal window:
```
pool/temperature/1 00.0
pool/temperature/2 00.0
```

### Configure Home Assistant

Add the following to your `configuration.yaml` file:

```
mqtt:
  broker: YOUR_POOL_PI_IP
  port: 1883
  client_id: pool
  username: mqtt
  password: YOUR_POOL_PI_PASSWORD
  
sensor:
  - platform: mqtt
    state_topic: "pool/temperature/1"
    name: "Pool"
    unit_of_measurement: "C"
  - platform: mqtt
    state_topic: "pool/temperature/2"
    name: "Deck"
    unit_of_measurement: "C"
```

#### Entities

![Entities](https://www.dropbox.com/s/64ked51hwj3wsnp/HA1.JPG?raw=1)

## Installation

### Pi / Relay

TODO