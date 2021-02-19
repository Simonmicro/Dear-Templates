---
summary: How you can use port forwarding even behind NAT (which is commonly used on mobile networks) and a guide for a classic VPN server for any amount of clients!
---

# What is this?
This little guide describes how you can use port forwarding even behind NAT (which is commonly used on mobile networks) and a guide for a classic VPN server for any amount of clients!

# Setup: Hardware
This guide requires (with some degree of variation) the following items:
* A VM somewhere with (fast) network access #sponsoredByEvolutionHost
* UniFi USG Pro 4
* A SIM card with an (unlimited) data plan, which is behind one or more NAT - so the noemal port forwarding won't work
* SIM card stick, I used [this](https://www.amazon.de/HUAWEI-E3372-Surfstick-microSD-Schwarz/dp/B011BRKPLE) which also has his own router firmware (so no special OS requirements) and does NAT. It also does not support port forwardings.
* Some old laptop with Linux, which passes the mobile network to the USB on a WAN port

# Setup: Software
OpenVPN - that's it. I tried to utilize Pritunl as it's easy to use and has a graphical UI, but it made only problems when I used it inside a docker environment - so it's possible to use it instead, but I opted to do everything from scratch anyways.

# How it is done
The used ips here:
* `10.8.0.0/24` VPN network
* `10.8.0.1` VMs IPv4 inside the VPN network
* `10.8.0.2` USGs IPv4 inside the VPN network
* `192.168.0.0/16` Network behind the USG
* `192.168.32.2` Webserver behind the USG

## Laptop (SIM modem)
This really depends on your specific stick and OS choice. Under Linux just insert the stick, wait until it assignes you an ip, open the webinterface and enter (& store) the pin and create a new network profile for the LAN port connected
with the USG. This should disable ipv6 when its not needed and select "share to other computers" for ipv4. Then make sure you check "connect when available", so it gets active on boot.

## VM
As first ensure the VM has ip(v4) forwarding enabled (otherwise the port forwarding service later on won't work):
```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.conf
```

### Certificates
Create a new file `/tmp/openssl.cnf` with:
```
[v3_ca]
basicConstraints = CA:FALSE
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
```
This is needed for signing CSRs with these enhancements. Use ONLY these `keyUsages`, the USG will fail when more are allowed...

Now create everything needed for the server (when you already have a CA, you may want to adapt this accordingly):
```bash
openssl dhparam -out dh2048.pem 2048
openvpn --genkey --secret ta.key

openssl genrsa -out ca.key 2048 # Generate new private key
openssl req -new -days 3650 -x509 -key ca.key -out ca.crt -subj '/CN=root-ca'
openssl genrsa -out server.key 2048
openssl req -new -key server.key -out server.csr -subj '/CN=server'
openssl x509 -req -days 3650 -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -extensions v3_ca -extfile /tmp/openssl.cnf
```

### OpenVPN server
Now intall `openvpn` and we prepare the `/etc/openvpn/server/server.conf`:
```
# Which TCP/UDP port should OpenVPN listen on?
port 1194

# TCP or UDP server (the latter has a lower overhead)?
proto udp

# When using UDP: Inform clients when the server is going down.
explicit-exit-notify 1

dev tun

# Server identity & encryption
;ca ca.crt
<ca>
[INSERT CONTENT OF ca.crt]
</ca>
;cert server.crt
<cert>
[INSERT CONTENT OF server.crt]
</cert>
;key server.key
<key>
[INSERT CONTENT OF server.key]
</key>

# Diffie hellman parameters.
;dh dh2048.pem
<dh>
[INSERT CONTENT OF dh2048.pem]
</dh>

topology subnet
server 10.8.0.0 255.255.255.0

# Store the dhcp ips for clients here...
ifconfig-pool-persist ipp.txt

# Read client specific settings from...
client-config-dir /etc/openvpn/ccd

keepalive 10 120

# For extra security beyond that provided by SSL/TLS, create an "HMAC firewall"
# to help block DoS attacks and UDP port flooding.
;tls-auth ta.key 0
<tls-auth>
[INSERT CONTENT OF ta.key]
</tls-auth>
key-direction 0

cipher AES-256-CBC

# Give up any special permissions on start (didn't work for me)...
;user nobody
;group nobody

# The persist options will try to avoid accessing certain resources on restart
# that may no longer be accessible because of the privilege downgrade.
persist-key
persist-tun

# Output a short status file showing current connections, truncated and rewritten
# every minute.
status openvpn-status.log

# Set the appropriate level of log
# file verbosity.
#
# 0 is silent, except for fatal errors
# 4 is reasonable for general usage
# 5 and 6 can help to debug connection problems
# 9 is extremely verbose
verb 3

# Allow older TLS versions to allow the USG to connect
tls-version-min 1.0

client-to-client

# Enter here the network behind the gateway, which you wan't to route to...
route 192.168.0.0 255.255.0.0
```

_NOTE_: Append the following to also route the default route and therefore allow internet access over the VPN:
```
push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS 1.1.1.1"
push "dhcp-option DNS 1.0.0.1"
```

Now activate the OpenVPN server (the stuff after the `@` is just passed trough to the `openvpn-server` service as argument):
```bash
sudo systemctl -f enable openvpn-server@server.service
sudo systemctl start openvpn-server@server.service
sudo systemctl status openvpn-server@server.service
```

### CCD: Client Specific Configs
To extend the configs of the connecting clients add inside the `/etc/openvpn/ccd` folder a new file named by the CN inside the clients certificate with:
```
# Configure static IP (obviuosly no two clients should get the same)
ifconfig-push 10.8.0.2 255.255.255.0
# Configure the routed network(s) - similar to the server.conf
iroute 192.168.0.0 255.255.0.0
```

### Port forwarding rules
Don't use `iptables-persistent`, it will also try to save `fail2ban` stuff (when installed)...

Create a new service under `/etc/systemd/system/forward_ports_to_vpn_clients.service`:
```systemd
[Unit]
Description=Enable port forwarding to OpenVPN client(s)
Requires=openvpn-server@server.service

[Service]
Type=simple
RemainAfterExit=true
Restart=on-failure
RestartSec=5s
ExecStart=/root/forward_ports_to_vpn_clients.sh start
ExecStopPost=/root/forward_ports_to_vpn_clients.sh stop

[Install]
WantedBy=multi-user.target
```

And the needed script under `/root/forward_ports_to_vpn_clients.sh`:
```bash
#!/bin/bash

start() {
    # Fail on unclean returns...
    set -e

    # Forward port 80 and 443 to the webserver behind the USG
    iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination 192.168.32.2:80
    iptables -t nat -A PREROUTING -p tcp --dport 443 -j DNAT --to-destination 192.168.32.2:443

    # Allow traffic back...
    iptables -t nat -A POSTROUTING -j MASQUERADE
}

stop() {
    # Remove all the previously added rules again (same commands; just with -D instead of -A)...
    iptables -t nat -D PREROUTING -p tcp --dport 80 -j DNAT --to-destination 192.168.32.2:80
    iptables -t nat -D PREROUTING -p tcp --dport 443 -j DNAT --to-destination 192.168.32.2:443
    iptables -t nat -D POSTROUTING -j MASQUERADE
}

case $1 in
  start|stop) "$1" ;;
esac
```

And enable the new service:
```bash
sudo chmod 700 /root/forward_ports_to_vpn_clients.sh
sudo systemctl enable forward_ports_to_vpn_clients.service
sudo systemctl start forward_ports_to_vpn_clients.service
sudo systemctl status forward_ports_to_vpn_clients.service
```

## Clients

### Certificates
Generate the set of certificates for the clients with (make sure to change the names of the files!):
```bash
openssl genrsa -out [CLIENT_USERNAME].key 2048
openssl req -new -key [CLIENT_USERNAME].key -out [CLIENT_USERNAME].csr -subj '/CN=[CLIENT_USERNAME]'
openssl x509 -req -days 3650 -in [CLIENT_USERNAME].csr -CA ca.crt -CAkey ca.key -CAcreateserial -out [CLIENT_USERNAME].crt
```

### Config(s)
This is the needed config file for every client - I recommend to first create a "test"-client (with a webserver to test the port forwarding stuff) before installing it to the USG!
```
client

dev tun

proto udp

# The hostname/IP and port of the server. You can have multiple remote
# entries to load balance between the servers.
remote jake.simonmicro.de 1194

resolv-retry infinite

# Most clients don't need to bind to a specific local port number.
nobind

# Give up any special permissions on start (didn't work for me)...
;user nobody
;group nobody

# The persist options will try to avoid accessing certain resources on restart
# that may no longer be accessible because of the privilege downgrade.
persist-key
persist-tun

# Client identity & encryption
;ca ca.crt
<ca>
[INSERT CONTENT OF ca.crt]
</ca>
;cert client.crt
<cert>
[INSERT CONTENT OF [CLIENT_USERNAME].crt]
</cert>
;key client.key
<key>
[INSERT CONTENT OF [CLIENT_USERNAME].key]
</key>
remote-cert-tls server
;tls-auth ta.key 1
<tls-auth>
[INSERT CONTENT OF ta.key]
</tls-auth>
key-direction 1
cipher AES-256-CBC

# Set the appropriate level of log
# file verbosity.
#
# 0 is silent, except for fatal errors
# 4 is reasonable for general usage
# 5 and 6 can help to debug connection problems
# 9 is extremely verbose
verb 3
```

## Install on USG
Upload the client config for the USG to `/config/user-data/client_vtun0.ovpn`.
Just execute the following commands:
```bash
configure
set interfaces openvpn vtun0 config-file /config/user-data/client_vtun0.ovpn
commit;save;exit
```
When you don't want to use the previously described port forwarding service, you could also change the primary interface for the port forwardinds on the USG itself. For that use `set port-forward wan-interface vtun0` right before
commiting. Note that then all port forwardings _only_ work on that interface!
Now export the config with `mca-ctrl -t dump-cfg > /tmp/config` and search it for the modified sections. For me it only was:
```json
{
    "interfaces": {
        "openvpn": {
            "vtun0": {
                "config-file": "/config/user-data/client_vtun0.ovpn"
            }
        }
    }
}
```
This part must now be [installed](https://help.ui.com/hc/en-us/articles/215458888-UniFi-How-to-further-customize-USG-configuration-with-config-gateway-json) into the `config.gateway.json` on the controller.
Also make sure to add some firewall rules to prevent the VM to go on a rampage - just in case...