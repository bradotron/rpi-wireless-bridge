# Raspberry Pi 4 Model B WiFi Ethernet Bridge
[Original](https://willhaley.com/blog/raspberry-pi-wifi-ethernet-bridge/)
[Will Haley](https://github.com/williamhaley)

# Option 1 - Same Subnet
This option is complex, but provides a more seamless experience. Bridged clients connected to the Pi should behave as if they were connected directly to the upstream network.

The following script configures everything in one go for a Pi with a standard Raspberry Pi OS install. This script is based off a very helpful Stack Overflow answer.

## Step 0: Connect to WiFi on the Pi like you normally would.

## Step 1: Save this script as a file named bridge.sh on your Pi.

```
#!/usr/bin/env bash

set -e

[ $EUID -ne 0 ] && echo "run as root" >&2 && exit 1

##########################################################
# You should not need to update anything below this line #
##########################################################

# parprouted  - Proxy ARP IP bridging daemon
# dhcp-helper - DHCP/BOOTP relay agent

apt update && apt install -y parprouted dhcp-helper

systemctl stop dhcp-helper
systemctl enable dhcp-helper

# Enable ipv4 forwarding.
sed -i'' s/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/ /etc/sysctl.conf

# Service configuration for standard WiFi connection. Connectivity will
# be lost if the username and password are incorrect.
systemctl restart wpa_supplicant.service

# Enable IP forwarding for wlan0 if it's not already enabled.
grep '^option ip-forwarding 1$' /etc/dhcpcd.conf || printf "option ip-forwarding 1\n" >> /etc/dhcpcd.conf

# Disable dhcpcd control of eth0.
grep '^denyinterfaces eth0$' /etc/dhcpcd.conf || printf "denyinterfaces eth0\n" >> /etc/dhcpcd.conf

# Configure dhcp-helper.
cat > /etc/default/dhcp-helper <<EOF
DHCPHELPER_OPTS="-b wlan0"
EOF

# Enable avahi reflector if it's not already enabled.
sed -i'' 's/#enable-reflector=no/enable-reflector=yes/' /etc/avahi/avahi-daemon.conf
grep '^enable-reflector=yes$' /etc/avahi/avahi-daemon.conf || {
  printf "something went wrong...\n\n"
  printf "Manually set 'enable-reflector=yes in /etc/avahi/avahi-daemon.conf'\n"
}

# I have to admit, I do not understand ARP and IP forwarding enough to explain
# exactly what is happening here. I am building off the work of others. In short
# this is a service to forward traffic from WiFi to Ethernet.
cat <<'EOF' >/usr/lib/systemd/system/parprouted.service
[Unit]
Description=proxy arp routing service
Documentation=https://raspberrypi.stackexchange.com/q/88954/79866
Requires=sys-subsystem-net-devices-wlan0.device dhcpcd.service
After=sys-subsystem-net-devices-wlan0.device dhcpcd.service

[Service]
Type=forking
# Restart until wlan0 gained carrier
Restart=on-failure
RestartSec=5
TimeoutStartSec=30
# clone the dhcp-allocated IP to eth0 so dhcp-helper will relay for the correct subnet
ExecStartPre=/bin/bash -c '/sbin/ip addr add $(/sbin/ip -4 -br addr show wlan0 | /bin/grep -Po "\\d+\\.\\d+\\.\\d+\\.\\d+")/32 dev eth0'
ExecStartPre=/sbin/ip link set dev eth0 up
ExecStartPre=/sbin/ip link set wlan0 promisc on
ExecStart=-/usr/sbin/parprouted eth0 wlan0
ExecStopPost=/sbin/ip link set wlan0 promisc off
ExecStopPost=/sbin/ip link set dev eth0 down
ExecStopPost=/bin/bash -c '/sbin/ip addr del $(/sbin/ip -4 -br addr show wlan0 | /bin/grep -Po "\\d+\\.\\d+\\.\\d+\\.\\d+")/32 dev eth0'

[Install]
WantedBy=wpa_supplicant.service
EOF

systemctl daemon-reload
systemctl enable parprouted
systemctl start parprouted dhcp-helper
```

## Step 2: Execute the script on your Pi like so.
```
sudo bash bridge.sh
```
## Step 3: Reboot.
```
sudo reboot
```

Done!

It may take a moment for your Pi to connect to WiFi, but once it does (and on subsequent reboots) it should be able to start forwarding traffic over the ethernet port.

# Option 2 - Separate Subnet
This option is simpler than the first option. However, this option results in a more limited setup.

Bridged clients will be on a separate subnet so the network configuration may not work like you expect. This option is fine if all you can about is connecting a bridged client to the Internet. Note that this script is a bit opinionated and chooses DNS servers and the subnet IP range for you. Update the script as needed.

## Step 0: Connect to WiFi on the Pi like you normally would.

## Step 1: Save this script as a file named bridge.sh on your Pi.

```
#!/usr/bin/env bash

set -e

[ $EUID -ne 0 ] && echo "run as root" >&2 && exit 1

apt update && \
  DEBIAN_FRONTEND=noninteractive apt install -y \
    dnsmasq netfilter-persistent iptables-persistent

# Create and persist iptables rule.
iptables -t nat -A POSTROUTING -o wlan0 -j MASQUERADE
netfilter-persistent save

# Enable ipv4 forwarding.
sed -i'' s/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/ /etc/sysctl.conf

# The Ethernet adapter will use a static IP of 169.69.1.1 on this new subnet.
cat <<'EOF' >/etc/network/interfaces.d/eth0
auto eth0
allow-hotplug eth0
iface eth0 inet static
  address 169.69.1.1
  netmask 255.255.255.0
  gateway 169.69.1.1
EOF

# Create a dnsmasq DHCP config at /etc/dnsmasq.d/bridge.conf. The Raspberry Pi
# will act as a DHCP server to the client connected over ethernet.
cat <<'EOF' >/etc/dnsmasq.d/bridge.conf
interface=eth0
bind-interfaces
server=8.8.8.8
domain-needed
bogus-priv
dhcp-range=169.69.1.50,169.69.1.150,12h
EOF

systemctl mask networking.service
```
## Step 2: Execute the script on your Pi like so.
```
sudo bash bridge.sh
```
## Step 3: Reboot.
```
sudo reboot
```
Done!

It may take a moment for your Pi to connect to WiFi, but once it does (and on subsequent reboots) it should be able to start forwarding traffic over the ethernet port.

Conclusion
You should now be able to connect a device to the ethernet port on the Raspberry Pi and receive an IP address.

I ran some very basic speed tests for my desktop connected through the Raspberry Pi bridge and was pleasantly surprised by the results.

I cannot guarantee how reliable this is for every possible use case, but it works well for me.