---
title: Preparing RaspberryPi for RedTeaming
published: true
---
# Preface

The RaspberryPi is an indispensable tool in a RedTeamer's arsenal. This microcomputer allows you to have constant access to the company's internal network without the physical presence of a hacker. The small body allows this tool to be invisible. However, this tool needs to be properly configured for a successful RedTeaming.

In this article, I will tell you what goals I set and how I set up my favorite tool to achieve these goals. It will be enough to run the script once on the device, and it will be completely ready for combat use.

# Goals

In order to successfully complete an engagement, I will highlight the main qualities that a microcomputer configured by me should have:

* Stable remote access to the microcomputer.
* Invisibility to the defense team.

# Tools used

I'm using a RaspberryPi 3 model B+, however everything I've said below is also true for other RaspberryPi models. We also need a USB-modem and a SIM-card with Internet access.

As an operating system, I will use Kali Linux version 2022.2 ([link here](https://kali.download/arm-images/kali-2022.2/kali-linux-2022.2-raspberry-pi-armhf.img.xz)), since it contains almost all the necessary software for the RedTeamer.

# Solution

So, there are two criteria that a microcomputer must meet, let's achieve them!

The scheme of the device looks something like this:
![Scheme](/resources/rpipreparing/2022-08-19%2013.58.55.jpg)

Further it will be clear why this is so, and not otherwise.

## Stable remote access to the microcomputer: solution

Stable access to the microcomputer in my model will be provided by the VPN tunnel to the C2 server via USB-modem. Only the ssh connection will go through the VPN tunnel. All other traffic must go over the Internet.

First we need to set up the VPN server. OpenVPN is best suited for this, since NetworkManager makes it very easy to raise VPN connections by ovpn file. I usually use [this script](https://github.com/Nyr/openvpn-install) for easy deploying OpenVPN server. 

In the future, we will need to remotely receive the .ovpn file from the server, for these purposes I wrote a small script that takes the name of the output configuration file as the first argument:
```bash
#!/bin/bash

client=$1

cd /etc/openvpn/server/easy-rsa/
EASYRSA_CERT_EXPIRE=3650 ./easyrsa build-client-full "$client" nopass

{
cat /etc/openvpn/server/client-common.txt
echo "<ca>"
cat /etc/openvpn/server/easy-rsa/pki/ca.crt
echo "</ca>"
echo "<cert>"
sed -ne '/BEGIN CERTIFICATE/,$ p' /etc/openvpn/server/easy-rsa/pki/issued/"$client".crt
echo "</cert>"
echo "<key>"
cat /etc/openvpn/server/easy-rsa/pki/private/"$client".key
echo "</key>"
echo "<tls-crypt>"
sed -ne '/BEGIN OpenVPN Static key/,$ p' /etc/openvpn/server/tc.key
echo "</tls-crypt>"
} > ~/"$client".ovpn


echo "Success! Configuration available in:" ~/"$client.ovpn"
exit
```
It's better to put this script in the home directory on the server, so that it can be easily called remotely later.

We have finished with the server part, now let's move on to setting up the RaspberryPi.

To achieve our goals, I will use NetworkManager, and more specifically its nmcli console utility. The reason for this is simple - the network must be managed from one source (or using one utility) so that there are no conflicts, you must either delete NetworkManager and do everything manually, or do everything by NetworkManager. Also NetworkManager is able to automatically setting up connections, which is very useful for achieving continuous access to the device.

First, we need to get the .ovpn configuration file in order to open the VPN tunnel to the C2 server. We can do this by calling the previous script over ssh. 

```bash
> ssh <user>@<hostname> -p <port> -t "sudo ./new_ovpn_client.sh <filename>; sudo cp /root/<filename>.ovpn /home/<user>/<filename>.ovpn;"
> scp -P <port> <user>@<hostname>:<filename>.ovpn .
```

Now you need to create a VPN connection on RaspberryPi:
```bash
> nmcli connection import type openvpn file <filename>.ovpn
```
Now, you need to make sure that this connection is raised automatically as soon as the USB modem is connected. NetworkManager allows you to do this. First, insert the USB modem into the microcomputer, wait until the connection is up (NetworkManager will automatically set it up) and look at the name from the list using next command. The name of the connection will be something like "eth1" or "Wired connection 1".
```bash
> nmcli connection show
```

Then make the VPN connection secondary to the USB modem.
```bash
> uuid=$(nmcli --get-values connection.uuid c show "<filename>")
> nmcli connection modify "<modem_connection_name>" +connection.secondaries $uuid
> systemctl restart NetworkManager
```

Now the VPN connection will only go through the USB-modem and will automatically set up as soon as the USB-modem is inserted into the RaspberryPi. 

## Invisibility to the defense team: solution

Since the only vector through which our device can be discovered is the Ethernet port, we need to make sure that nothing else goes through Ethernet port. It is desirable not to raise the connection on it at all for as long as possible. 

To do this, disable routes on this interface.

But first we need to make eth0 managed by NetworkManager. This can be achieved by deleting the eth0 file from the /etc/network/interfaces.d folder
```bash
> mv /etc/network/interfaces.d/eth0 ~/eth0
> systemctl restart NetworkManager
> nmcli connection add type ethernet con-name eth0 ifname eth0
```
Done. Now eth0 is managed by NetworkManager. Now let's do what we originally wanted: disable the default routes and set connection down on eth0 in order to prevert detection by BlueTeam. 
```bash
> nmcli connection modify "eth0" ipv4.never-default yes
> nmcli connection modify "eth0" ipv6.never-default yes
> nmcli connection modify "eth0" ipv4.ignore-auto-routes yes
> nmcli connection modify "eth0" ipv6.ignore-auto-routes yes
> nmcli connection down eth0
> systemctl restart NetworkManager
```

Default mac-address of RaspberryPi makes it easy to detect this device, so we need to change mac-address on Ethernet port.

We can do it by NetworkManager:
```bash
> nmcli connection modify eth0 802-3-ethernet.cloned-mac-address <new_mac_address>
```
A similar situation with hostname:
```bash
> hostnamectl set-hostname xerox-printer
> sed -i '1d' /etc/hosts
> sed -i "1s/^/127.0.1.1       xerox-printer\n/" /etc/hosts
> systemctl restart networking
```

Excellent! We've just got the RaspberryPi ready for a RedTeaming. Congratulations!


I also created a script that automates all actions, link [here](https://gist.github.com/inagen/d6b8f17bd6ed12f367924c5aaaabd553). However, before running it, you must remove all connections except for the modem (I will fix it in future).

# Conclusion

NetworkManager is a great tool for creating a combat image of the operating system for the RaspberryPi. The resulting configuration is easily replicable (just run the script on a new device) and quite reliable. This image has been tested many times in real and artificial conditions and has never let me down.
