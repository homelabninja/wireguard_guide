# installing WireGuard


You can find the related youtube video on this link
[Youtube](https://www.youtube.com/watch?v=0x9wyN-mNOI&t=1s)


## Introduction to WG
Why Wiguard

1. Speed
2. fast
3. less code less memory usage


# Setup Server

## Step 1: update
Ubuntu setup

```bash
sudo su
apt update

apt upgrade
```

## Step 2: Find public IP

finding out the public ip



go to google and find your public ip or 
```bash
curl ifconfig.me
```




## Step 3: install necessary packages

install necessary packages


```bash

apt install iptables net-tools nano wireguard -y
```

verify that wireguard module and related tools are installed on your machine

```bash
wg --version
```

## step 4: create server keys
move into `/etc/wireguard` directory. 
create server public and private keys

```bash
wg genkey | tee server_privatekey | wg pubkey > server_publickey
```

Take note of server public and private keys. Dont share the private key. anyone with the key can decrypt your traffic. 


## step 5: Create Server configuration file
Once public and private keys are generated, create a new file in the same location called `wg0.conf`. This is a default file for your first wireguard interface and it is a convention to use names like wg0, wg1 etc. 

```bash
nano wg0.conf
```
This will open the nano editor and create the file. 

Write the following content into the configuration file  

run the following command to find your local ip address and network interface

```bash
ifconfig
```
This will list your interface and local IP.
our VPN server will have the VPN IP address of 11.0.0.1
```ini
[Interface]
## Address : A private IP address for wg0 interface.
Address = 11.0.0.1/24
## Specify the listening port of WireGuard, I like port 51820 udp, you can change it.
ListenPort = 51820
## A privatekey of the server ( cat /etc/wireguard/privatekey)
PrivateKey = <server_privatekey>
SaveConfig = true
## The PostUp will run when the WireGuard Server starts the virtual VPN tunnel.
## The PostDown rules run when the WireGuard Server stops the virtual VPN tunnel.
## Specify the command that allows traffic to leave the server and give the VPN clients access to the Internet. 

PostUp = iptables -A FORWARD -i wg0 -j ACCEPT
PostUp = iptables -t nat -A POSTROUTING -o <YOUR_NETWORK_INTERFACE> -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT
PostDown = iptables -t nat -D POSTROUTING -o <YOUR_NETWORK_INTERFACE> -j MASQUERAD

```


Replace the interface with your own interface. 
Be sure to replace the server private key as well in the above example. 
The other commands simply mean to forward network traffic.

optionally you may need to change the wg0.conf mode


```bash
chmod 600 wg0.conf
```


## step 6: Enable IP forarding
Next we will enable IP forwarding



open the `etc/sysctl.conf` using `nano` editor. 


uncomment the ip forwarding line
```bash
net.ipv4.ip_forward=1
```

enabling ip forwarding allows packets to route between internet and VPN clients. 
press CTRL+S to save and CTRL+X to exit.
apply changes using the following command
```bash
sysctl -p
```
## step 7 configure firewall

Sometimes it is necessary to configure Firewall. 

install firewall application if it is not already installed

```bash
sudo apt install ufw
```

check the status of the firewall

```bash
ufw status
```


Note if you are using ssh to access the vpn server, allow port 22 as well, otherwise you would lose SSH access

```bash
sudo ufw allow 22/tcp
sudo ufw allow 51820/udp
```


Now the firewall will enable to allow traffic on 51820/udp port

```bash
ufw reload
```


## step 8: Enable wireguard server

finally start and enable wireguard server
This will run the server automatically at startup


```bash
systemctl start wg-quick@wg0
systemctl enable wg-quick@wg0
```

To checkout the status of the wireguard server, 

```bash
systemctl status wg-quick@wg0
```



## Step 9: Creating a Client

Create can be any machine running wireguard, be it windows or Linux

On the linux client make sure that wireguard is installed.
```bash
sudo apt-get install wireguard -y
```


Lets try to create a Linux client, as this step will be quite similar to setup of the server


```bash
wg genkey | tee client_privatekey | wg pubkey > client_publickey
```

now create the wg0.conf file for client as well.

```ini
# NOTE: This config file should be added to the client and not the server

[interface]
# This is the config for my linux laptop
PrivateKey = <client_priavate_key>
Address = 11.0.0.2/8
 # this is the VPN ip of this client

[Peer]

###Public of the WireGuard VPN Server
PublicKey = <server_private_key>
# public key for wireguard vpn is same
### IP and Port of the WireGuard VPN Server
Endpoint = <SERVER_PUBLIC_IP>:51820
### Allow only 11.0.0.x through VPN
AllowedIPs = 11.0.0.1/8
# only route traffic directed to this network through vpn
Persistentkeepalive = 25
# keep connection alive, ping every 25 seconds. 
```

next we need to register the client in the server
for this we will use the public key of the client. Take note of the key and go to your client machine .
also take note of the VPN IP of the client. 
you will use these two values to run a command on server that will register the respective client to it. 

use the `set` command to register the new peer.

```bash
wg set wg0 peer <client_public_key> allowed-ips <vpn-ip-of-client>
```

once this command is run , the client is registered on the server,

Next you just need to establish connection from client to server

to run the wireguard client on client,

```bash
wg-quick up wg0
```


wireguard is client initiated protocol so the communication must be started by the client. 

The connection should be established now

To test the connection, from client machine, 
```bash

ping 11.0.0.1 
# ping the vpn server
```
you should see a response from the server

