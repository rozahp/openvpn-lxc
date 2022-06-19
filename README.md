
## Installing OpenVPN server in a LXC container

## 1. Create a container

	lxc launch ubuntu:20.04 openvpn
	lxc exec openvpn -- bash
	apt-get update

## 2. Configure the firewall

	ufw allow OpenSSH
	ufw enable

## 3. Install Easy-RSA & Openvpn

	apt-get install openvpn easy-rsa
	su -l ubuntu
	mkdir ~/easy-rsa
	ln -s /usr/share/easy-rsa/* ~/easy-rsa/
	sudo chown ubuntu ~/easy-rsa
	chmod 700 ~/easy-rsa

## 4. Generate PKI for OpenVPN

	cd ~/easy-rsa
	nano vars

	set_var EASYRSA_ALGO "ec"
	set_var EASYRSA_DIGEST "sha512"

	./easyrsa init-pki

## 5. Creating an OpenVPN server certificate

	cd ~/easy-rsa
	./easyrsa gen-req server nopass
	sudo cp /home/ubuntu/easy-rsa/pki/private/server.key /etc/openvpn/server/

## 6. Build Easy-RSA CA

	cd ~/
	mkdir easy-rsa-ca
	ln -s /usr/share/easy-rsa/* ~/easy-rsa-ca/
	chmod 700 /home/ubuntu/easy-rsa-ca
	cd ~/easy-rsa-ca
	./easyrsa init-pki
	nano vars

	set_var EASYRSA_REQ_COUNTRY    "US"
	set_var EASYRSA_REQ_PROVINCE   "NewYork"
	set_var EASYRSA_REQ_CITY       "New York City"
	set_var EASYRSA_REQ_ORG        "DigitalOcean"
	set_var EASYRSA_REQ_EMAIL      "admin@example.com"
	set_var EASYRSA_REQ_OU         "Community"
	set_var EASYRSA_ALGO           "ec"
	set_var EASYRSA_DIGEST         "sha512"

	./easyrsa build-ca

## 7. Signing the OpenVPN server certificate

	cp /home/ubuntu/easy-rsa/pki/reqs/server.req .
	./easyrsa import-req server.req server
	./easyrsa sign-req server server

	sudo cp pki/issued/server.crt /etc/openvpn/server
	sudo cp pki/ca.crt /etc/openvpn/server

## 8. Configuring OpenVPN cryptographic material

	cd ~/easy-rsa
	openvpn --genkey --sercert ta.key
	sudo cp ta.key /etc/openvpn/server

## 9. Generating a client certificate and key pair

	cd 
	mkdir -p ~/client-configs/keys
	chmod -R 700 ~/client-configs
	cd ~/easy-rsa
	./easyrsa gen-req client1 
	cp pki/private/client1.key ~/client-configs/keys
	cp pki/reqs/client1.req ~/easy-rsa-ca
	cd ~/easy-rsa-ca
	./easyrsa import-req client1.req client1
	./easyrsa sign-req client client1
	cp pki/issued/client1.crt ~/client-configs/keys/
	cp ~/easy-rsa/ta.key ~/client-configs/keys/
	sudo cp /etc/openvpn/server/ca.crt ~/client-configs/keys/
	sudo chown ubuntu:ubuntu ~/client-configs/keys/*

## 10. Configuring OpenVPN

	sudo cp /usr/share/doc/openvpn/examples/sample-config-files/server.conf.gz /etc/openvpn/server/
	nano /etc/openvpn/server/server.conf

	local 10.x.x.x  # OpenVPN Server IP 
	port 1194
	proto udp
	dev tun
	ca ca.crt
	cert server.crt
	key server.key  # This file should be kept secret
	dh none
	topology subnet
	server 10.8.0.0 255.255.255.0
	ifconfig-pool-persist /var/log/openvpn/ipp.txt
	push "route 10.x.x.0 255.255.255.0" # Subnet of the local LXD container
	push "route 192.168.x.0 255.255.255.0" # Subnet of local network
	push "redirect-gateway def1 bypass-dhcp"
	push "dhcp-option DNS a.a.a.a"	# DNS server 1
	push "dhcp-option DNS b.b.b.b"	# DNS server 2
	keepalive 30 120
	tls-auth ta.key 0
	key-direction 0 #CUSTOM
	cipher AES-256-GCM
	auth SHA256
	max-clients 5
	user nobody
	group nogroup
	persist-key
	persist-tun
	status /var/log/openvpn/openvpn-status.log
	log-append  /var/log/openvpn/openvpn.log
	verb 1
	mute 20
	explicit-exit-notify 0

## 11. Adjusting the OpenVPN server networking configuration

	sudo nano /etc/sysctl.conf
	net.ipv4.ip_forward = 1
	sudo sysctl -p

## 12. Firewall configuration

	sudo nano /etc/ufw/before.rules

	#
	# rules.before
	#
	# Rules that should be run before the ufw command line added rules. Custom
	# rules should be added to one of these chains:
	#   ufw-before-input
	#   ufw-before-output
	#   ufw-before-forward
	#

	# START OPENVPN RULES
	# NAT table rules
	*nat
	:POSTROUTING ACCEPT [0:0]
	# Allow traffic from OpenVPN client to eth0 (change to the interface you discovered!)
	-A POSTROUTING -s 10.8.0.0/8 -o eth0 -j MASQUERADE
	COMMIT
	# END OPENVPN RULES
	
	# Don't delete these required lines, otherwise there will be errors
	*filter

	sudo nano /etc/default/ufw

	DEFAULT_FORWARD_POLICY="ACCEPT"

	sudo ufw allow 1194/udp
	sudo ufw allow OpenSSH
	sudo ufw disable 
	sudo ufw enable

## 13. Starting OpenVPN

	sudo systemctl -f enable openvpn-server@server.service
	sudo systemctl start openvpn-server@server.service

## 14. Creating the client configuration infrastructure

	mkdir -p ~/client-configs/files
	cp /usr/share/doc/openvpn/examples/sample-config-files/client.conf ~/client-configs/base.conf)
	
	nano ~/client-configs/base.conf

	client
	dev tun
	proto udp
	remote myserver.mydomain.tld 1194 # Domain name of your OpenVPN server
	resolv-retry infinite
	nobind
	persist-key
	persist-tun
	mute-replay-warnings
	remote-cert-tls server
	cipher AES-256-GCM
	auth SHA256
	verb 1
	mute 20
	key-direction 1

	nano ~/client-configs/make_config.sh

	#!/bin/bash

	# First argument: Client identifier

	KEY_DIR=~/client-configs/keys
	OUTPUT_DIR=~/client-configs/files
	BASE_CONFIG=~/client-configs/base.conf

	cat ${BASE_CONFIG} \
	    <(echo -e '<ca>') \
	    ${KEY_DIR}/ca.crt \
	    <(echo -e '</ca>\n<cert>') \
	    ${KEY_DIR}/${1}.crt \
	    <(echo -e '</cert>\n<key>') \
	    ${KEY_DIR}/${1}.key \
	    <(echo -e '</key>\n<tls-auth>') \
	    ${KEY_DIR}/ta.key \
	    <(echo -e '</tls-auth>') \
	    > ${OUTPUT_DIR}/${1}.ovpn

	# Comment: does not work with <tls-crypt> from iPhone APP

	chmod 700 ~/client-configs/make_config.sh

## 15. Generating client configurations

	cd ~/client-configs
	./make_config.sh client1
	cp ~/client-configs/files/client1.ovpn to your iPhone or what-ever ...

## 16. Fix error 1

	1. sudo systemctl edit openvpn-server@

	2. Add following text

	[Service]
	LimitNPROC=infinity

	3. sudo systemctl daemon-reload

	4. sudo systemctl start openvpn-server@server.service

## 17. Fix error 2

	1. <tls-auth> in client1.ovpn NOT <tls-cert>

## EOF
