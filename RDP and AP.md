# Raspberry headless config

## Remote Desktop Access

1. Clean install Raspberry Pi OS lite version and update with
    
    `sudo apt update && sudo apt upgrade && sudo apt autoremove`
    
2. Install a GUI on top of Raspberry Pi OS Lite. In order to have a GUI, we need these 4 things:
    - Display Server
    - Desktop Environment
    - Window Manager
    - Login Manager
    
    ### Display Server
    
    `sudo apt-get install --no-install-recommends xserver-xorg`
    
    ### DE, Window and Login Manager
    
    If we choose Raspberry Pi Desktop, Window and Login managers are also installed:
    
    `sudo apt-get install raspberrypi-ui-mods`
    
3. Install Remote Desktop Protocol server
    
    `sudo apt-get install xrdp`
    
    We select to boot without auto-login either in console or in GUI. If we want to also enable VNC server, then we have to choose to boot in GUI.
    
    ```jsx
    raspi-config > system options > boot
    ```
    

## Access Point and WIFI client

To configure the Pi as both AP and client we have to use a WIFI dongle that supports this combination. We can check that with `iw list`

![comb.png](assets/comb.png)

The above output shows that we can use the second configuration but we are restricted on using the same WIFI channel both for the managed and AP. 

This means that we will not be able to connect to a WIFI with different channel and maintain the AP enabled, unless we manually modify the AP channel.

### WIFI client

1.  Configure the Pi to access a WIFI network as a client (station)

 `sudo vim /etc/wpa_supplicant/wpa_supplicant.conf`

```bash
country=GR
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1

network={
    ssid="SSID_OF_NETWORK"
    psk="NETWORK_PASSWORD"
}
```

1. Edit the interfaces configuration
    
    `sudo vi /etc/network/interfaces`
    
    An example is shown below. The `uap0` interface will be used later on for our Access Point.
    
    ```bash
    source-directory /etc/network/interfaces.d
    
    auto lo
    auto eth0 # If there is an ethernet adapter
    auto wlan0
    auto uap0
    
    iface eth0 inet dhcp
    iface lo inet loopback
    
    allow-hotplug wlan0
    
    iface wlan0 inet dhcp
    wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf
    
    iface uap0 inet static
      address 192.168.50.1
      netmask 255.255.255.0
      network 192.168.50.0
      broadcast 192.168.50.255
      gateway 192.168.50.1
    ```
    
    Reboot and check if `wlan0` is configured correctly with `ifconfig`
    

### Access Point

1. Install [hostapd](https://w1.fi/hostapd/) and [dnsmasq](http://www.thekelleys.org.uk/dnsmasq/doc.html) for the AP mode
    
     `sudo apt-get install hostapd dnsmasq`
    
2. Create a new *hostapd* configuration `sudo vim /etc/hostapd/hostapd.conf`
    
    ```bash
    interface=uap0
    ssid=testPiAP
    hw_mode=g
    channel=11
    macaddr_acl=0
    auth_algs=1
    ignore_broadcast_ssid=0
    wpa=2
    wpa_passphrase=badpassword
    wpa_key_mgmt=WPA-PSK
    wpa_pairwise=TKIP
    rsn_pairwise=CCMP
    ```
    
    We have to use the same channel as the connected Wifi. This can be found using 
    
    `iwlist wlan0 channel` or `iw dev`
    
3.  Modify the *hostapd* default configuration to use our new configuration file
`sudo vim /etc/default/hostapd`
    
    by adding
    `DAEMON_CONF="/etc/hostapd/hostapd.conf"`
    
4. Create a script to set the interface to AP mode, start *hostapd* and set some *iptables* configuration.
`sudo vim /usr/local/bin/hostapdstart`
    
    ```bash
    #!/bin/bash
    PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
    iw dev wlan0 interface add uap0 type __ap
    ifup uap0 # mallon
    service dnsmasq restart
    sysctl net.ipv4.ip_forward=1
    iptables -t nat -A POSTROUTING -s 192.168.50.0/24 ! -d 192.168.50.0/24 -j MASQUERADE
    ifup uap0
    hostapd /etc/hostapd/hostapd.conf
    ```
    
    `chmod 775 /usr/local/bin/hostapdstart`
    
    ### DNS Setup
    
5. We need the new AP to hand out IP addresses to our clients. We enable AP interface with  `ifup uap0` and configure *dnsmasq* `sudo vim /etc/dnsmasq.conf`
    
    ```bash
    interface=lo,uap0
    no-dhcp-interface=lo,wlan0
    bind-interfaces
    server=8.8.8.8
    domain-needed
    bogus-priv
    dhcp-range=192.168.50.50,192.168.50.150,12h
    ```
    
6. Start *dnsmasq service*
`sudo service dnsmasq start`

### Startup

1. Edit the rc.local script to run hostapdstart on boot up
`sudo vi /etc/rc.local`
    
    ```bash
    /bin/bash /usr/local/bin/hostapdstart
    ```
    
    and reboot
