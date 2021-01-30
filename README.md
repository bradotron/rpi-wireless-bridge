# rpi-wireless-bridge

# Setup
1. These steps presume that the raspberry pi is setup to operate headless, via ssh, and is connected to wifi. Wifi setup should be in /etc/wpa_supplicant/wpa_supplicant.conf
```
network={
        ssid="mynetwork"
        psk="secret"
        key_mgmt=WPA-PSK
}
``` 

# Step 1: Update the Raspberry Pi
```
sudo apt-get update && sudo apt-get upgrade -y && sudo apt-get install rpi-update dnsmasq -y
```
This also installs dnsmasq, which will be used later.

Restart the Raspberry Pi
```
sudo shutdown -r now
```

# Step 2: Setup Ethernet Static IP
1. edit the interfaces file
```
sudo nano /etc/network/interfaces
```
Add the following
```
allow-hotplug eth0  
iface eth0 inet static  
    address 192.168.2.1
    netmask 255.255.255.0
    network 192.168.2.0
    broadcast 192.168.2.255
```
* change the address/network to whatever you prefer

# Step 3: dnsmasq Setup
Save the old dnsmasq conf file.
```
sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.orig  
sudo nano /etc/dnsmasq.conf
```
Add this to dnsmasq.conf:
```
interface=eth0
listen-address=192.168.2.1
bind-interfaces  
server=8.8.8.8 # Forward DNS requests to Google DNS  
domain-needed  
bogus-priv  
dhcp-range=192.168.2.50,192.168.2.150,12h
```

# Step 4: Enable IPv4 forwarding
```
sudo nano /etc/sysctl.conf
```
Uncomment the following:
```
net.ipv4.ip_forward=1
```

# Step 5: IP Tables
Enter this:
```
sudo iptables -t nat -A POSTROUTING -o wlan0 -j MASQUERADE
```
Configure it to load on reboot by first saving to a file
```
sudo sh -c "iptables-save > /etc/iptables.ipv4.nat"
```
Then create a 'hook' file with a line to restore the ip tables
```
sudo nano /lib/dhcpcd/dhcpcd-hooks/70-ipv4-nat
```
add this line into the file:
```
iptables-restore < /etc/iptables.ipv4.nat
```

# Step 6: Reboot
After a reboot, your pi should connect to your wifi, and anything plugged into the pi's ethernet port will also have internet access.