# Complete OpenVPN Server Setup Guide for beginners on AWS EC2 Instance 

This guide provides detailed, step-by-step instructions for beginners to set up an OpenVPN server on AWS EC2 with creating multiple users and multiple devices connection configurations.

## Prerequisite: Launch an EC2 Instance ( Create free AWS account using Credit/Debit card to eligible for many Free tier services including EC2 Computing)

1. Log into AWS Management Console
2. Choose your preferred server location (country) from the top right corner — ideally, select the nearest country for faster connectivity.
3. Navigate to EC2 dashboard
4. Click "Launch instance"
5. Select Ubuntu Server 24.04 LTS
6. Choose t2.micro instance type (free tier eligible)
7. Configure security group with these inbound rules:
   - SSH (port 22) from your IP
   - Custom UDP (port 1194) from anywhere (0.0.0.0/0)
     or can change the port number but then make sure to use that port in configuration files
   - Custom UDP (port 1195) from anywhere (0.0.0.0/0) 
8. Create or select a key pair which you named/generated "xxxx.pem"
9. Launch the instance
10. Set the downloaded .pem file permissions:
   ```
   chmod 400 xxxx.pem
   ```
## Part 1: Initial Setup and Connection

 Step 1: Connect to your EC2 instance from your local computer
 - Crate a seperate folder and save your key in that folder and open the terminal of Linux in that folder and connect your EC2 instance via SSH
```bash
 ssh -i "xxxx.pem" ubuntu@YOUR_EC2_PUBLIC_IP
```
Replace YOUR_EC2_PUBLIC_IP with your actual EC2 public IP or DNS name.

Step 2: Update the system

```
sudo apt update
sudo apt upgrade -y
```
Step 3: Install OpenVPN and Easy-RSA
```
sudo apt install openvpn easy-rsa -y
```
## Part 2: Certificate Authority Setup

Step 4: Set up the Easy-RSA directory
```
mkdir -p ~/easy-rsa
```
```
ln -s /usr/share/easy-rsa/* ~/easy-rsa/
```
```
cd ~/easy-rsa
```
Step 5: Initialize the PKI infrastructure
```
./easyrsa init-pki
```
Step 6: Build the Certificate Authority
```
./easyrsa build-ca nopass
```
When prompted, enter a Common Name (e.g., "myvpn"). This is an identifier for your CA.

## Part 3: Server Certificate and Key Generation

Step 7: Generate the server certificate request and key
```
cd ~/easy-rsa
```
```
./easyrsa gen-req server nopass
```
Press Enter to accept the default name "server".

Step 8: Sign the server certificate
```
cd ~/easy-rsa
```
```
./easyrsa sign-req server server
```

Type "yes" when prompted to confirm.

Step 9: Generate Diffie-Hellman parameters

```
cd ~/easy-rsa
```
```
./easyrsa gen-dh
```
This will take a few minutes to complete.

Step 10: Generate the TLS authentication key
```
cd ~/easy-rsa
```
```
openvpn --genkey --secret pki/ta.key
```
## Part 4: Server Configuration

Step 11: Create server directory and copy files
```
sudo mkdir -p /etc/openvpn/server
```
```
sudo cp ~/easy-rsa/pki/ca.crt /etc/openvpn/server/
```
```
sudo cp ~/easy-rsa/pki/issued/server.crt /etc/openvpn/server/
```
```
sudo cp ~/easy-rsa/pki/private/server.key /etc/openvpn/server/
```
```
sudo cp ~/easy-rsa/pki/dh.pem /etc/openvpn/server/
```
```
sudo cp ~/easy-rsa/pki/ta.key /etc/openvpn/server/
```

Step 12: Create the log directory
```
sudo mkdir -p /var/log/openvpn/
```

Step 13: Create the first server configuration file
```
sudo nano /etc/openvpn/server/server.conf
```
Copy and paste the following configuration:
```
port 1194
proto udp
dev tun
ca ca.crt
cert server.crt
key server.key
dh dh.pem
auth SHA512
tls-auth ta.key 0
topology subnet
server 10.8.0.0 255.255.255.0
ifconfig-pool-persist /var/log/openvpn/ipp.txt
push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS 1.1.1.1"
push "dhcp-option DNS 1.0.0.1"
push "block-outside-dns"
push "dhcp-option DOMAIN-ROUTE ."
keepalive 10 120
cipher AES-256-CBC
user nobody
group nogroup
persist-key
persist-tun
status /var/log/openvpn/status.log
verb 3
explicit-exit-notify 1
duplicate-cn
```
Save and exit: Press Ctrl+X, then Y, then Enter.

Note: IPs are private IPs and you can choose any Private IP but Good alternative choices would be:

   • Any 10.x.y.z/24 subnet (where x, y, and z are between 0-255) 
    
   • 172.16.x.y/24 through 172.31.x.y/24 
    
  • 192.168.x.y/24 
    
Just make sure your chosen subnets don't overlap with the network where your client devices connect from.

## Part 5: Network Configuration

Step 14: Enable IP forwarding
```
sudo nano /etc/sysctl.conf
```
Find the line that says #net.ipv4.ip_forward=1 and remove the # to uncomment it:

net.ipv4.ip_forward=1

If the line doesn't exist, add it at the end of the file.

Save and exit: Press Ctrl+X, then Y, then Enter.

Apply the changes:
```
sudo sysctl -p
```
Step 15: Configure firewall rules

First, identify your network interface:
```
ip route | grep default
```
Note the interface name (e.g ens3 /replace it with your aws network interface). Now configure iptables:

```
sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o ens3 -j MASQUERADE
sudo iptables -A INPUT -i ens3 -p udp --dport 1194 -j ACCEPT
sudo iptables -A INPUT -i tun+ -j ACCEPT
sudo iptables -A FORWARD -i tun+ -j ACCEPT
sudo iptables -A FORWARD -i tun+ -o ens5 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i ens5 -o tun+ -m state --state RELATED,ESTABLISHED -j ACCEPT
```

Step 16: Make firewall rules persistent
```
sudo apt install iptables-persistent -y
```
When prompted, select "Yes" to save the current IPv4 and IPv6 rules.

If you make changes to iptables after installation, save the rules with:
```
sudo netfilter-persistent save
```
```
sudo netfilter-persistent reload
```
## Part 6: Client Configuration Setup

Step 17: Create directories for client configurations
```
mkdir -p ~/client-configs/files
```
```
chmod 700 ~/client-configs/files
```

Step 18: Create the base client configuration file
```
nano ~/client-configs/base.conf
```
Add the following content (replace YOUR_SERVER_IP with your actual EC2 public IP):
```
client
dev tun
proto udp
remote YOUR_SERVER_IP 1194
resolv-retry infinite
nobind
persist-key
persist-tun
remote-cert-tls server
auth SHA512
cipher AES-256-CBC
ignore-unknown-option block-outside-dns
verb 3
```
Save and exit: Press Ctrl+X, then Y, then Enter.

Step 19: Create client configuration generation script
```
nano ~/client-configs/make_config.sh
```
Add the following content:

```
#!/bin/bash

# First argument: Client identifier

KEY_DIR=~/easy-rsa/pki
OUTPUT_DIR=~/client-configs/files
BASE_CONFIG=~/client-configs/base.conf

cat ${BASE_CONFIG} \
    <(echo -e '<ca>') \
    ${KEY_DIR}/ca.crt \
    <(echo -e '</ca>\n<cert>') \
    ${KEY_DIR}/issued/${1}.crt \
    <(echo -e '</cert>\n<key>') \
    ${KEY_DIR}/private/${1}.key \
    <(echo -e '</key>\n<tls-auth>') \
    ${KEY_DIR}/ta.key \
    <(echo -e '</tls-auth>\nkey-direction 1') \
    > ${OUTPUT_DIR}/${1}.ovpn

```
Save and exit: Press Ctrl+X, then Y, then Enter.

Step 20: Make the script executable
```
chmod 700 ~/client-configs/make_config.sh
```
## Part 7: Start the First OpenVPN Server

Step 21: Start and enable the OpenVPN service
```
sudo systemctl start openvpn-server@server.service
```
```
sudo systemctl enable openvpn-server@server.service
```
Step 22: Check the service status
```
sudo systemctl status openvpn-server@server.service
```
Press q to exit the status display.

## Part 8: Create First User (Multiple Devices Allowed)

Step 23: Generate user1 certificate and key
```
cd ~/easy-rsa
```
```
./easyrsa gen-req user1 nopass
```
Press Enter to accept the default name.

Step 24: Sign user1 certificate
```
cd ~/easy-rsa
```
```
./easyrsa sign-req client user1
```
Type "yes" when prompted.

Step 25: Generate user1 configuration file

```
cd ~/client-configs
```
```
./make_config.sh user1
```
Check the available configuration files
```
ls -la ~/client-configs/files/
```
Download the .ovpn files to your local computer to get connected with VPN.
From your local terminal/new seperate terminal (not from the SSH connected terminal to EC2) run this command :
```
scp -i "xxxx.pem" ubuntu@YOUR_EC2_PUBLIC_IP:~/client-configs/files/user1.ovpn ./
```
Here replace xxxx.pem with your actual key and YOUR_EC2_PUBLIC_IP with your actual EC2 public IP.

## Use the .ovpn files with OpenVPN clients

For Windows:
        ◦ Install OpenVPN GUI 
        ◦ Right-click the OpenVPN GUI icon and run as administrator 
        ◦ Right-click the system tray icon 
        ◦ Select "Import file" and choose your .ovpn file which you downloaded 
        ◦ Click "Connect" to establish the VPN connection 
For macOS:
        ◦ Install Tunnelblick 
        ◦ Double-click the .ovpn file 
        ◦ Follow prompts to import the configuration 
        ◦ Click "Connect" button 
For Android:
        ◦ Install OpenVPN Connect from the Play Store 
        ◦ Tap the + icon 
        ◦ Select "Import" and locate your .ovpn file 
        ◦ Tap "Add" and then "Connect" 
For iOS:
        ◦ Install OpenVPN Connect from the App Store 
        ◦ Share the .ovpn file to the OpenVPN app 
        ◦ Tap "Add" when prompted 
        ◦ Tap the toggle to connect 
        
Verify the IP location and connection DNS server for any potential DNS leaks using https://dnsleaktest.com/ 

Using this guide, you can create .ovpn files and secure your connection via OpenVPN across multiple devices using a single configuration file, in accordance with EC2 limitations.
Note: This setup is ideal for general browsing and secure connectivity, as AWS allows 100GB of data transfer per month in free tier limit. Exceeding this limit will incur additional charges per GB.
