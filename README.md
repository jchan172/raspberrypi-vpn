# Raspberry Pi VPN Server

## Server Side Setup

1. [Set up NOOBS and install Raspbian](https://www.raspberrypi.org/help/noobs-setup/). To make life easier, make sure to enable SSH by running `raspi-config` and then navigating to Advanced Options, SSH, then select "Enable". Y

2. Update Raspberry Pi

		sudo apt-get update
		sudo apt-get upgrade

3. Install software

		sudo apt-get install openvpn vim

4. Start doing stuff as root

		sudo su

5. Copy OpenVPN's easy-rsa directory as template files (easy-rsa is for authorization to connect to VPN server)
		cp –r /usr/share/doc/openvpn/examples/easy-rsa/2.0 /etc/openvpn/easy-rsa 

6. Open `vars` file for editing. These are easy-rsa parameter settings.

		vim /etc/openvpn/easy-rsa/vars

7. Change the EASY_RSA variable to:

		export EASY_RSA="/etc/openvpn/easy-rsa"

8. Build CA Certificate and Root CA certificate. 

	Load `vars`

		cd /etc/openvpn/easy-rsa
		source ./vars

	**Careful here!** Now you'll remove any previous keys, if there are any. If you have keys you don’t want to remove in this folder (like you’re doing this tutorial a second time), skip this command. 

		./clean-all

	Build the certificate authority.

		./build-ca

9. After `./build-ca`, the Raspberry Pi is going to prompt for a bunch of optional fields for you to fill out if you want to: Country Name, State or Province Name, Locality Name, Organization Name, Organizational Unit, Common Name, Name, and Email Address. If you don't care to fill out these fields, just hit “enter” each instance to have the Pi fill in the default value. Below is what it looks like:

		root@raspberrypivpn:/etc/openvpn/easy-rsa# ./build-ca
		Generating a 1024 bit RSA private key
		..........................++++++
		......++++++
		writing new private key to 'ca.key'
		-----
		You are about to be asked to enter information that will be incorporated
		into your certificate request.
		What you are about to enter is what is called a Distinguished Name or a DN.
		There are quite a few fields but you can leave some blank
		For some fields there will be a default value,
		If you enter '.', the field will be left blank.
		-----
		Country Name (2 letter code) [US]:US
		State or Province Name (full name) [CA]:WA
		Locality Name (eg, city) [SanFrancisco]:Seattle
		Organization Name (eg, company) [Fort-Funston]:Chan
		Organizational Unit Name (eg, section) [changeme]:
		Common Name (eg, your name or your server's hostname) [changeme]:chanvpn
		Name [changeme]:
		Email Address [mail@host.domain]:

10. Build the key server.

		./build-key-server [your_server_name] # example: ./build-key-server chanvpn

	Once again, there will be prompts to enter optional fields, but pay attention to 3 fields:
	
	1. Common Name MUST be the server name you picked. It should default to this.
	2. A challenge password? MUST be left blank.
	3.  `Sign the certificate? [y/n]` Obviously, you must type “y”
	
	You’ll get a message that says the certificate will be certified for 3,650 more days. So basically if you use your VPN long enough, you’ll have to do this process again in 10 years.
	
	`1 out of 1 certificate requests certified, commit? [y/n]` Obviously, type “y.”

	Output should look like: 

		Certificate is to be certified until Jun 18 22:51:59 2025 GMT (3650 days)
		Sign the certificate? [y/n]:y

		1 out of 1 certificate requests certified, commit? [y/n]y
		Write out database with 1 new entries
		Data Base Updated

11. We're done with the server side stuff. Now it's time to build keys for users.

		./build-key-pass [your-user-name] # example: ./build-key-pass jack

	- Enter PEM pass phrase Make it a password you will remember! It asks you to input this twice, so there’s no danger of ruining it. 
	- A challenge password? MUST be left blank.
	- `Sign the certificate? [y/n]` Enter "y". Signing certifies it for 10 more years.

12. Apply des3 encryption to the key. des3 is a complex encryption algorithm that's applied three times to each data block to keep hackers from breaking through it with brute force. OpenSSL stands for an open source implementation of Secure Socket Layer, a standard method of setting up a secure connection. **You need to perform steps 11 and 12 for every client you set up.**

		cd /etc/openvpn/easy-rsa/keys
		openssl rsa -in jack.key -des3 -out jack.3des.key  

		# when it asks "Enter pass phrase for jack.key", just enter the one used before

13. Generate Diffie-Hellman key exchange. This is the central code that makes your VPN server tick, an exchange that lets two entities with no prior knowledge of one another share secret keys over a public server. Like RSA, it’s one of the earliest cryptosystems out there.

		cd /etc/openvpn/easy-rsa/
		./build-dh # this will take like 5 minutes 

14. Finally, generate a static HMAC key to be used later when creating client configuration file. This is used to prevent DOS attacks. With this in place, the server won't even entertain the idea of authenticating an access request unless it detects this static key first. Thus, a hacker can’t just spam the server with random repeated requests. 

		openvpn –-genkey –-secret keys/ta.key

15. Create a `server.conf` file for server to use as configuration.

		vim /etc/openvpn/server.conf 
		# see the server.conf provided

16. Edit another file to allow server to forward internet traffic.

		vim /etc/sysctl.conf
		# uncomment the line that says: net.ipv4.ip_forward=1

17. Apply changes. This command configures kernel parameters at runtime. The -p tells it to reload the file with the changes you just made. 

		sysctl -p 

18. Raspbian has a firewall to protect your Raspberry Pi from unknown and unexpected Internet sources. We still want the firewall to protect us from most incoming and outgoing network traffic, but we need to poke an OpenVPN-shaped hole in the firewall. Raspbian’s firewall configuration resets by default when you reboot the Pi. We want to make sure it remembers the OpenVPN connection is always permitted, so what we’re going to do is create a simple script which runs on boot:

		vim /etc/firewall-openvpn-rules.sh

	With these contents (remember to change 192.168.XX.X to whatever Raspberry Pi's address is):

		#!/bin/sh 

		iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j SNAT --to-source 192.168.XX.X

	Explanation: 10.8.0.0 is the default address for Raspberry Pi for clients that are connected to the VPN. "eth0" stands for ethernet port. Switch this to "wlan0" if you’re on a wireless connection, which is not recommended. 

19. Make the script run in the `interfaces` setup on boot.

		vim /etc/network/interfaces

	Add the following after `iface eth0 inet dhcp` (note there is a tab before `pre-up`): 

			pre-up /etc/firewall-openvpn-rules.sh

	See `interfaces` file for the change

20. Reboot the Raspberry Pi.

		sudo reboot

	Note that there will be errors like:

		[warn] Kernel lacks cgroups or memory controller not available, not starting cgroups. ... (warning).
		[FAIL] Not running dhcpcd because /etc/network/interfaces ... failed!
		[FAIL] defines some interfaces that will use a DHCP client ... failed!
		.
		.
		.
		[warn] startpar: service(s) skipped, program is not configured: dhcpcd ... (warning).

	Despite these failures and warnings, the VPN server will run just fine.

## Client Side Setup

With the previous steps finished, you've got a functional VPN server ready on your Raspberry Pi. Keys have been created too, but clients still have no way to connect to the server. They will need a configuration file with their keys in order to connect. Also, at this point, you'll need to use a dynamic DNS service so that you can access your Raspberry Pi from outside the local network.

1. Your public IP address can change, unless your ISP gives a static IP address. So, you'll need a dynamic DNS provider to provide a stable name that maps to your changing IP address (there will be code/cron job on the server that updates the dynamic DNS provider when the IP changes). A good, free dynamic DNS provider is Duck DNS at [www.duckdns.com](www.duckdns.com). Sign up on the website (asks you to use Facebook, Google, Twitter, or Reddit). You can then choose a subdomain (http://<SUBDOMAIN>.duckdns.com). Finally, click on the install tab, click on "pi" for the operating system option, and follow the instructions. The instructions creates a cron job that updates Duck DNS so that it knows what your public IP address is. It periodically checks the IP address and updates Duck DNS.

2. Enter the router config (go to 192.168.1.1 or whatever your router is) and forward a port. You'll need to create the port forwarding rule, specifying your Raspberry Pi as the IP address to forward and then for the application to forward, select "Custom Port". Select "UDP" as the protocol, source ports as "Any", destination ports as "1194", forward to port as "Same as Incoming Port", schedule as "always", and WAN connection type as "All Broadband Devices". This makes it so that any traffic that hits the 1194 port (OpenVPN's official port number) will be sent to the Raspberry Pi (the VPN server).

3. Back on the Raspberry Pi, create default settings to generate configuration files for each client.

		vim /etc/openvpn/easy-rsa/keys/Default.txt

	With the following:

		client
		dev tun
		proto udp
		remote <INSERT YOUR DYNAMIC DNS ADDRESS> 1194
		resolv-retry infinite
		nobind
		persist-key
		persist-tun
		mute-replay-warnings
		ns-cert-type server
		key-direction 1
		cipher AES-128-CBC
		comp-lzo
		verb 1
		mute 20

3. Create script to generate the config file that clients will use, and then execute script.

		vim /etc/openvpn/easy-rsa/keys/makeOVPN.sh
		# see makeOVPN.sh provided
		chmod 700 makeOVPN.sh
		./makeOVPN.sh

	You should see something like:

		root@raspberrypivpn:/etc/openvpn/easy-rsa/keys# ./makeOVPN.sh
		Please enter an existing Client Name:
		jack
		Client’s cert found: jack
		Client’s Private Key found: jack.3des.key
		CA public Key found: ca.crt
		tls-auth Private Key found: ta.key
		Done! jack.ovpn Successfully Created.

4. Now, you'll need to copy the configuration files (the ones with `.ovpn` extension) from `/etc/openvpn/easy-rsa/keys/` over to the clients. For OSX, use a program like Fugu to SCP files over or access the files via SFTP. On Windows, use something like WinSCP. In order to access the files, you'll need to enable access of the folder:

		chmod 777 -R /etc/openvpn

	Be sure to put back restriction after you're done so that other people can't get keys.

		chmod 600 -R /etc/openvpn

5. Now you can use client software to connect to the VPN server! For your PC, Android, or iOS mobile device, download [OpenVPN Connect](https://openvpn.net/). On OSX, TunnelBlick is a good, free option. To use TunnelBlick, install it, and then double-click a `.ovpn` file (should default to open TunnelBlick). If everything's been done correctly, you'll be asked to enter a keyphrase (enter in the one that you used to create the client key), and you'll be connected!