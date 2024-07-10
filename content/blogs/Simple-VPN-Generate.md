---
title: "各常見 VPN 架設簡易教學"
date: 2024-07-10T11:00:00+08:00
draft: false
github_link: ""
author: "Chumy"
tags:
  - Networking
  - Infra
image: ""
description: ""
toc: 
---

## IPsec
### Install strongswan
``` bash
apt install strongswan strongswan-pki libcharon-extra-plugins libcharon-extauth-plugins libstrongswan-extra-plugins
```

### Config file
* /etc/ipsec.conf
    * for connect config
* /etc/ipsec.secrets
    * A config to save your password, preshare key, private key
* /etc/ipsec.d/
    * The directorys where the certificates or private keys is placed.
        * /etc/ipsec.d/cacerts/
            * for CA certificate
        * /etc/ipsec.d/certs/
            * for certificate
        * /etc/ipsec.d/private/
            * for private key
    * Certificate for IPsec must have **EKU**:
        * **IP security IKE intermediate (oid is 1.3.6.1.5.5.8.2.2)**

### Simple config of ipsec.conf
``` 
conn <connect name>
        # Auto mean do you want to actively initiate a connection or passively accept a connection?
        auto=<add | start>	
        type=<tunnel | transport>
        keyexchange=<ike | ikev1 | ikev2>

        #left mean local
        left=<left ip addres default is %any>
        leftauth=<server auth method>
        leftsubnet=<what subnet that client can forward whan client connect to vpn>

        #right mean remote
        right=<right ip addres default is %any>
        rightauth=<client auth method>
        rightsourceip=<if you use tunnel mode. What subnet do you want to assign to client>

        # etc.
```

### Simple config of ipsec.secrets
```
# for public auth
: RSA "<private key file name>"

# for password auth
<username> : EAP "<password>"

# for preshare key auth
<remote ip> : PSK "<preshare key>" 
```

## Wireguard
### Setup by command(Linux)
- Add interface
    - `ip link add dev wg0 type wireguard`
- Setup ip
    - `ip address add dev wg0 192.168.2.1/24`
    - `ip address add dev wg0 192.168.2.1 peer 192.168.2.2`
- Setup wg configurations
    - `wg setconf wg0 myconfig.conf`
    - `wg set wg0 listen-port 51820 private-key /path/to/private-key peer ABCDEF... allowed-ips 192.168.88.0/24 endpoint 209.202.254.14:8172`
- Start interface
    - `ip link set up dev wg0`

### Setup by configuration
- Configuration file
    - /etc/wireguard/wg0.conf
- Start interface
    - `systemctl enable wg-quick@wg0`
    - `wg-quick up wg0`

#### Example connfigurations - Client
![](https://i.imgur.com/5fzLxqW.png)

#### Example connfigurations - Server
![](https://i.imgur.com/1NujUxH.png)

### Interface
- Address (optional)
    - IP address and netmask of the interface
- ListenPort
    - Wg service listen port
- PrivateKey
    - Private key of the interface
- PreUp / PreDown / PostUp / PostDown
    - Run shell scripts before / after interface up / down
    - E.g.
        - Setup firewall rules

### Peer
- PublicKey
    - Public key of the peer
- AllowedIPs
    - IP addresses that are allowed to pass through this peer
- Endpoint (Optional)
    - Location of the peer
    - Wg will also use the previous connections to detect this configuration
- PersistentKeepalive (Optional)
    - By default, Wg send packs only if there are data to be send
    - Send packs to peer periodically to bypass NAT or Firewall
- PresharedKey (Optional)
    - Pre-shared key for additional symmetric encryption

### Generate Key Pair
- Key pair
    - wg genkey > privatekey
    - wg pubkey < privatekey > publickey
- Pre-shared key
    - wg genpsk > preshared

## OpenVPN
### Configuration
- Template directory
    - You can choose your config Template in this directory
    - /usr/share/doc/openvpn/examples/sample-config-files/
    - ![](https://i.imgur.com/fkheOLQ.png)
- Write your config file in /etc/openvpn/

### Simple server config
Your server certificate need **EKU**: ***server authentication***
![](https://i.imgur.com/ESCembk.png)

### Simple client config
Your client certificate need **EKU**: ***client authentication***
![](https://i.imgur.com/PPViaX9.png)

- You can put your CA, certificate and private in the config with `<ca></ca> <cert></cert> <key></key>` 
- But don’t use it with `ca, cert, key` attributes in the same time.
    - `ca <CA certificate path>`
    - `cert <certificate path>`
    - `key <private key path>`

![](https://i.imgur.com/n865ryu.png)

### Enable and Start
#### Start your openvpn server or client
``` bash
systemctl start openvpn@<vpn config name without .conf>
```
#### Start at boot
``` bash
systemctl enable openvpn@<vpn config name without .conf>
```

### User-authentication
1. Simply by signing client certs.
2. Use Username/password
3. Use 3rd party authentication
    - RADIUS
    - LDAP

#### Server side
##### PAM authentication
``` ovpn
plugin /usr/lib/openvpn/plugins/openvpn-plugin-auth-pam.so login
```

##### Use a shell script to auth
``` ovpn
auth-user-pass-verify <script path> via-env
script-security 3 # To allow script reading passwords
```

#### Client side

##### Auth
``` ovpn
auth-user-pass
```
##### Auth with specify user
###### in config
``` ovpn
auth-user-pass <user file>
```
###### in `<user file>`
```
<Username>
<Password>
```


