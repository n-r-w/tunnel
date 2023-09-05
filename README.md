# Shadowsocks, Cloak, Tun2socks configuration

WARNING: This configuration has problems with UDP traffic (voip, games, etc). The problem is still unclear.

## Data is transmitted through a chain:
- Computer in the local network
- Tun2socks on the local server (forwards all traffic to Shadowsocks)
- Shadowsocks on the local server (forwards all traffic to Cloak)
- Cloak-client on the local server (creates a tunnel disguised as HTTPS traffic)
- Cloak-server on the remote server (receives traffic from Cloak-client)
- Shadowsocks on the remote server (decrypts data from the local Shadowsocks and sends it to the internet)

If you want, you can exclude cloak from the chain and connect the Shadowsocks client to the Shadowsocks server directly. To do this, you will need to open port 20220 on the server and use remote_server:20220 instead of 127.0.0.1:1984 in the local Shadowsocks configuration. But I recommend not to do this, because shadowsocks traffic is detected by provider's filters as "unknown" and may be blocked. Cloak traffic is detected as https which is always allowed. The only vulnerability is if all your traffic goes through one host, then it will look suspicious.

Because of laziness, I did everything under root and in the /root/tunel directory, but you can change the directory to your own.

## Tested on:
- Remote server Ununtu 22.04 (VPS, 1 CPU core, 1GB RAM)
- Local server Ubuntu 20.04 (2 CPU cores, 2GB RAM, single ethernet port). 
- Speedtest Download Mbps: 363.33, Upload Mbps: 218.68
- The first connection (for example, after starting a computer configured for a gateway with Tun2socks) takes 2-5 seconds. There are no delays in further work. This is because cloak does not support a persistent connection by default, making it difficult to identify the server as a proxy.

Tested on following versions:
- shadowsocks-go: https://github.com/database64128/shadowsocks-go/releases/tag/v1.8.0
- tun2socks: https://github.com/xjasonlyu/tun2socks/releases/tag/v2.5.1
- cloak: https://github.com/cbeuw/Cloak/releases/tag/v2.7.0

## Install shadowsocks on remote server
1. Create folder ```/root/tunel/```
2. Download latest release from https://github.com/database64128/shadowsocks-go, make it executable ```chmod +x ./shadowsocks-go``` and copy to ```/root/tunel/```
3. Generate unique password and save it
4. Create shadowsocks-go config and copy to ```/root/tunel/shadowsocks-go-server.json```
```
{
    "servers": [
        {
            "name": "ss-2022",
            "listen": ":20220",
            "protocol": "2022-blake3-aes-128-gcm",
            "enableTCP": true,
            "listenerTFO": true,
            "enableUDP": true,
            "mtu": 1500,
            "psk": "base64 encoded password from step 3",
            "uPSKStorePath": ""
        }
    ]
}
```
5. Create systemd entry ```/etc/systemd/system/shadowsocks-go.service```
```
[Unit]
Description=Shadowsocks Go Proxy Platform
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/root/tunel/shadowsocks-go -confPath /root/tunel/shadowsocks-go-server.json -zapConf systemd
ExecReload=/usr/bin/kill -USR1 $MAINPID

[Install]
WantedBy=multi-user.target
```
```
systemctl daemon-reload
systemctl enable shadowsocks-go
systemctl start shadowsocks-go
# check result
systemctl status shadowsocks-go
```
## Install cloak on remote server
1. Download latest server release from https://github.com/cbeuw/Cloak, make it executable ```chmod +x ./ck-server``` and copy to ```/root/tunel/```
2. Generate public/private keys ```./ck-server -key``` and save it
3. Generate uid ```./ck-server -uid``` and save it
4. Create cloak server config and copy to ```/root/tunel/ckserver.json```
```
{
  "ProxyBook": {
    "shadowsocks": [
      "tcp",
      "127.0.0.1:20220"
    ]
  },
  "BindAddr": [
    ":443",
    ":80"
  ],
  "BypassUID": [
    "uid from step 3"
  ],
  "RedirAddr": "github.com",
  "PrivateKey": "private key from step 2"
}
```
5. Create systemd entry ```/etc/systemd/system/ck-server.service```
```
[Unit]
Description=Cloak server
After=shadowsocks-go.service
Wants=network-online.target

[Service]
ExecStart=/root/tunel//ck-server -c /root/tunel/ckserver.json
Restart=always

[Install]
WantedBy=multi-user.target
```
```
systemctl daemon-reload
systemctl enable ck-server
systemctl start ck-server
# check result
systemctl status ck-server
```
6. Open http ports https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-ubuntu-22-04
```
ufw allow http
ufw allow https
```

## Configure local server
- Configure static IP address for local server: https://www.freecodecamp.org/news/setting-a-static-ip-in-ubuntu-linux-ip-address-tutorial. Make sure this address does not overlap with the DHCP address range.
- Or you can use default DHCP and configure the DHCP server to a permanent address for this server.
- Enable ip forward: ```nano /etc/sysctl.conf```, set net.ipv4.ip_forward=1, apply changes ```sysctl -p```

## Install cloak on local server
1. Create folder ```/root/tunel/```
2. Download latest client release from https://github.com/cbeuw/Cloak, make it executable ```chmod +x ./ck-client``` and copy to ```/root/tunel/```
3. Create cloak client config and copy to ```/root/tunel/ck-client.json```
```
{
  "Transport": "direct",
  "ProxyMethod": "shadowsocks",
  "EncryptionMethod": "plain",
  "UID": "BypassUID from server",
  "PublicKey": "public key from server",
  "ServerName": "github.com",
  "AlternativeNames": ["cloudflare.com", "bing.com", "stackoverflow.com"],
  "NumConn": 100,
  "BrowserSig": "chrome",
  "StreamTimeout": 300
}
```
4. Create systemd entry ```/etc/systemd/system/ck-client.service```
```
[Unit]
Description=CK Server
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/root/tunel/ck-client -c /root/tunel/ck-client.json -s remote_server_ip
Restart=always

[Install]
WantedBy=multi-user.target
```
```
systemctl daemon-reload
systemctl enable ck-client
systemctl start ck-client
# check result
systemctl status ck-client
```

## Install shadowsocks on local server
1. Download latest release from https://github.com/database64128/shadowsocks-go, make it executable ```chmod +x ./shadowsocks-go``` and copy to ```/root/tunel/```
2. Create shadowsocks-go config and copy to ```/root/tunel/shadowsocks-go-client.json```. 1984 - default cloak port. So the shadowsocks client thinks it's connecting to the shadowsocks server, but it's actually a cloak that will then forward the traffic to the remote shadowsocks
```
{
    "servers": [
        {
            "name": "socks5",
            "listen": "127.0.0.1:1080",
            "protocol": "socks5",
            "enableTCP": true,
            "listenerTFO": true,
            "enableUDP": true,
            "mtu": 1500
        },
        {
            "name": "http",
            "listen": ":8080",
            "protocol": "http",
            "enableTCP": true,
            "listenerTFO": true
        }
    ],
    "clients": [
        {
            "name": "ss-2022",
            "protocol": "2022-blake3-aes-128-gcm",
            "endpoint": "127.0.0.1:1984",
            "enableTCP": true,
            "dialerTFO": true,
            "enableUDP": true,
            "mtu": 1500,
            "psk": "base64 encoded password from shadowsocks server configuration"
        }
    ]
}

```
3. Create systemd entry ```/etc/systemd/system/shadowsocks-go.service```
```
[Unit]
Description=Shadowsocks Go Proxy Platform
After=cloak-client.service
Wants=network-online.target

[Service]
ExecStart=/root/tunel/shadowsocks-go -confPath /root/tunel/shadowsocks-go-client.json -zapConf systemd
ExecReload=/usr/bin/kill -USR1 $MAINPID

[Install]
WantedBy=multi-user.target
```
```
systemctl daemon-reload
systemctl enable shadowsocks-go
systemctl start shadowsocks-go
# check result
systemctl status shadowsocks-go
```

## install tun2socks on local server
1. Download latest release from https://github.com/xjasonlyu/tun2socks, make it executable ```chmod +x ./tun2socks``` and copy to ```/root/tunel/```
2. Create startup script, copy to ```/root/tunel/tun2socks.sh``` and make it executable ```chmod +x ./tun2socks.sh```
```
# Find out the name of the network adapter
ip a
```
script:
```
#!/bin/bash
ip tuntap add mode tun dev tun0
ip addr add 198.18.0.1/15 dev tun0
ip link set tun0 up
ip route del default via 198.18.0.1
ip route del default via local_network_gateway # example ip route del default via 192.168.1.1
ip route add default via 198.18.0.1 dev tun0 metric 1
ip route add default via local_network_gateway dev network_adapter metric 10 # example: ip route add default via 192.168.1.1 dev enp1s0 metric 10
ip route add remote_server_ip/32 via local_network_gateway # example: ip route add remote_server_ip/32 via 192.168.1.1
/root/tunel/tun2sock -device tun0 -proxy socks5://127.0.0.1:1080 -interface network_adapter # example: /root/tunel/tun2sock -device tun0 -proxy socks5://127.0.0.1:1080 -interface enp1s0
ip route del default via 198.18.0.1
ip route del default via local_network_gateway # example: ip route del default via 192.168.1.1
ip route del remote_server_ip/32
ip route add default via local_network_gateway dev network_adapter # example ip route add default via 192.168.1.1 dev enp1s0
```
3. Create systemd entry ```/etc/systemd/system/tun2socks.service```
```
[Unit]
Description=tun2socks
After=ck-client.service shadowsocks-go.service
Wants=network-online.target

[Service]
ExecStart=/root/tunel/tun2socks.sh

[Install]
WantedBy=default.target
```
```
systemctl daemon-reload
systemctl enable tun2socks
systemctl start tun2socks
# check result
systemctl status tun2socks
```

## LAN setup
In the DHCP server settings, set the gateway to the address of the tun2socks server (from ```Configure local server``` step) for machines from which traffic should go through the tunnel
