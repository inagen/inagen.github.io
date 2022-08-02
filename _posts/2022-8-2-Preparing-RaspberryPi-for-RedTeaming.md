---
title: Preparing RaspberryPi for RedTeaming
published: true
---

# Goals

RaspberryPi is one of the main instruments in the RedTeamer's arsenal. But for a successful deployment, you need to properly prepare it. Probably, you need to create an image of the operating system for everyday use.

Here is list of requirements that a working image should satisfy in my opinion:
* Replicability.
* Invisibility to other devices on the local network.
* Continuous remote access to the device.

The most obvious solution looks like this: a USB modem is connected to the raspberry, through which the raspberry is connected to the VPN server. Based on the above requirements and the fact that VPN-traffic is very often controlled in companies we define the conditions to be followed.

* VPN traffic should go ONLY through usb modem.
* Any other traffic must go through the modem, but not through the VPN.
* Nothing goes through ethernet.

# Solution
## Server-side
To achieve the above goals, it is best to use NetworkManager and its nmcli console utility, as it is best integrated into the system. And we will use kali-linux-2022.2-raspberry-pi-armhf as the OS.

First of all, we need to deploy our VPN server. For setting up OpenVPN server I usually use [this](https://github.com/Nyr/openvpn-install). Also we need to get the .ovpn configuration file remotely from our device. For this reason, I made a small script:

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

This script will be located in the home directory on the server, in the future it will be executed by clients during the configuration of the operating system image.


## Client-side

Let's move on to what will be executed on the client side. For client-side, I also wrote my own script:
```bash
#!/bin/bash

while getopts u:h:p: flag
do
    case "${flag}" in
        u) user=${OPTARG};;
        h) host=${OPTARG};;
        p) port=${OPTARG};;
    esac
done

if [[ -z $user || -z $host || -z $port || "$EUID" -ne 0 ]]
then
	echo "Usage: sudo ./script.sh -u <username> -h <host> -p <port>";
	exit 1;
fi

echo "Receiving ovpn file from server..."
name=$(echo $RANDOM | md5sum | head -c 25;)
ssh $user@$host -p $port -t "sudo ./new_ovpn_client.sh $name; sudo cp /root/$name.ovpn /home/$user/$name.ovpn;"
scp -P $port $user@$host:$name.ovpn .


# Returns connection name by interface
nm_connection_of() {
    # $1 = name of network interface to query
    con_name=$(nmcli -g GENERAL.CONNECTION device show "$1")
    if [ "$con_name" = "" ]; then
        echo "ERROR: no connection associated with $1" >&2
        return 1
    fi
    echo "$con_name"
}

echo "Importing VPN connection to NetworkManager..."
nmcli connection import type openvpn file $name.ovpn
uuid=$(nmcli --get-values connection.uuid c show "$name")
nmcli connection modify "$(nm_connection_of eth1)" +connection.secondaries $uuid
echo "VPN connection imported succesfully!"
echo;

echo "Setting eth0 managed by NetworkManager..."
mv /etc/network/interfaces.d/eth0 ~/eth0
systemctl restart NetworkManager
nmcli connection add type ethernet con-name eth0 ifname eth0
echo "eth0 is managed by NetworkManager now!"
echo;

echo "Waiting for NetworkManager proper restart..."
sleep 10
echo;

echo "Disabling default routes for eth0..."
nmcli connection modify "$(nm_connection_of eth0)" ipv4.never-default yes
nmcli connection modify "$(nm_connection_of eth0)" ipv6.never-default yes
nmcli connection modify "$(nm_connection_of eth0)" ipv4.ignore-auto-routes yes
nmcli connection modify "$(nm_connection_of eth0)" ipv6.ignore-auto-routes yes
nmcli connection down eth0
echo "Default routes for eth0 are disabled!"
echo;

echo "Disabling default routes for tun0..."
nmcli connection modify "$(nm_connection_of tun0)" ipv4.never-default yes
nmcli connection modify "$(nm_connection_of tun0)" ipv6.never-default yes
nmcli connection modify "$(nm_connection_of tun0)" ipv4.ignore-auto-routes yes
nmcli connection modify "$(nm_connection_of tun0)" ipv6.ignore-auto-routes yes
echo "Default routes for tun0 are disabled!"
echo;

echo "Disabling default routes for vpn..."
nmcli connection modify $uuid ipv4.never-default yes
nmcli connection modify $uuid ipv6.never-default yes
nmcli connection modify $uuid ipv4.ignore-auto-routes yes
nmcli connection modify $uuid ipv6.ignore-auto-routes yes
echo "Default routes for vpn are disabled!"
echo;

echo "Restarting NetworkManager..."
systemctl restart NetworkManager
echo "NetworkManager restarted!"
echo;

echo "Preparation finished!"

```
Let's go through it piece by piece.

Firstly we get the .ovpn configuration file from the server. The next piece of code does just that. The file name is `$name.ovpn`. Where `$name` is a random string of 25 characters.
```bash
echo "Receiving ovpn file from server..."
name=$(echo $RANDOM | md5sum | head -c 25;)
ssh $user@$host -p $port -t "sudo ./new_ovpn_client.sh $name; sudo cp /root/$name.ovpn /home/$user/$name.ovpn;"
scp -P $port $user@$host:$name.ovpn .
```

Once the ovpn file has been received, a connection will be created using nmcli.
```bash
nmcli connection import type openvpn file $name.ovpn
uuid=$(nmcli --get-values connection.uuid c show "$name")
nmcli connection modify "$(nm_connection_of eth1)" +connection.secondaries $uuid
```
Also in this piece of code we say that the vpn connection will raised as soon as the connection is raised via the usb modem.  Note: the ethernet port on the RaspberryPi always has an eth0 interface, and the usb port takes the eth1 interface (when a modem is connected).

By default, NetworkManager can't manage eth0, the following part fixes this problem:
```bash
mv /etc/network/interfaces.d/eth0 ~/eth0
systemctl restart NetworkManager
nmcli connection add type ethernet con-name eth0 ifname eth0
``` 

At this moment, we have achieved continuous remote access to the device and full control over the eth0 interface. However, we want no traffic to go through the vpn tunnel and eth0. The rest of the script helps to achieve this: it says that we don't want default routes for eth0 and VPN connection.

After executing this script, the device is completely ready for use in real conditions. But don't forget to change the hostname and MAC-address of the device :D

# Conclusion
NetworkManager is a great tool for creating a combat image of the operating system for the RaspberryPi. The resulting configuration is easily replicable (just run the script on a new device) and quite reliable. This image has been tested many times in real and artificial conditions and has never let me down.