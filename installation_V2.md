# Custom Cloud VPN Server Project: Installation Documentation

## [1. Initial Server Setup with Ubuntu 20.04](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-20-04)

### 1.0 Prereq
* Deactivated DigitalOcean account and create a new one
* Deploy a new Droplet to server as the Certificate Authority (CA) Server
    * Droplet IP address: 138.68.93.182
    * Default root password: ITM684.Vy

### 1.1 Log in as root
* `ssh root@138.68.93.182`
    * Enter root password (ITM684.Vy) when prompted 

### 1.2 Create a new user
* `adduser vy`
    * Enter password: 26072002
* Press *Enter* to skip unnecessary fields
* Press *Y* and *Enter*

### 1.3 Grant administrative privileges
* `usermod -aG sudo vy`
    * To allow the normal user to run commands with administrative privileges 

### 1.4 Set up a basic firewall
**HAVE TO CHANGE THE SSH PORT**
* `ufw app list`
    * The UFW firewall will make sure only connections to certain service are allowed
* `ufw allow OpenSSH`
    * To make sure the firewall allows SSH connections 
* `ufw enable`
    * Enable the firewall
* Press *Y* and *Enter*
* `ufw status`
    * To see if SSH connections are still allowed
    * Currently, the firewall is blocking all connections except for SSH so if you want to install and configure additional services, the firewall settings need to be adjusted 

### 1.5 Enabling external access for your regular user
* `ssh vy@138.68.93.182`
    * To SSH into your user account
* Enter the regular user's password: 26072002

***

## [2. Setting up and Configuring a Certificate Authority (CA) on Ubuntu 20.04](https://www.digitalocean.com/community/tutorials/how-to-set-up-and-configure-a-certificate-authority-ca-on-ubuntu-20-04)

### 2.1 Install Easy-RSA
* easy-rsa is a CA management tool used to generate a private key and public root certificate 
* The public root certificate is used to sign requests from clients and servers
* `sudo apt update`
* `sudo apt install easy-rsa`
    * Note: make sure that you are logged in as the non-root sudo user 
* Press *Y* and *Enter*

### 2.2 Prepare a Public Key Infrastructure (PKI) directory
* Note: make sure that you are logged in as the non-root sudo user and do not use sudo to run any of the following commands
* `mkdir ~/easy-rsa`
* `ln -s /usr/share/easy-rsa/* ~/easy-rsa/`
    * Create a symbolic link to the easy-rsa package files
    * This is better than copying the easy-rsa package files diretcory into the PKI directory because any updates to the easy-rsa package will be automatically reflected
* `chmod 700 /home/vy/easy-rsa`
    * To retrict access to the PKI directory and make it so only the owner can access it
* `cd ~/easy-rsa`
* `./easyrsa init-pki`
    * To initialize the PKI inside the easy-rsa directory 
    * Newly created PKI directory: /home/vy/easy-rsa/pki

### 2.3 Create a CA
* Make sure that you're in the easy-rsa directory, if not:
    * `cd ~/easy-rsa`
* `nano vars`
    * Create and edit a file called vars 
* Paste the following lines into the file:
    ```
    ~/easy-rsa/vars
    set_var EASYRSA_REQ_COUNTRY    "US"
    set_var EASYRSA_REQ_PROVINCE   "Hawaii"
    set_var EASYRSA_REQ_CITY       "Manoa"
    set_var EASYRSA_REQ_ORG        "ITM684"
    set_var EASYRSA_REQ_EMAIL      "bvt@hawaii.edu"
    set_var EASYRSA_REQ_OU         "Community"
    set_var EASYRSA_ALGO           "ec"
    set_var EASYRSA_DIGEST         "sha512"
    ```
* Press *CTRL+X* to exit
* Press *Y* and *Enter*
* `./easyrsa build-ca`
    * To create the root public and private key pair for the CA
    * Ran into an error stating:
        ```
        Easy-RSA error:
        Conflicting 'vars' files found.
        Priority should be given to your PKI vars file:
        * /home/vy/easy-rsa/pki/vars
        ```
    * Solve error by making a copy of the vars folder outside of the pki, naming it vars, then removing the old vars folder
        ```
        cd easy-rsa
        cp -r vars vars1
        rm -r vars
        ```
* CA Key Passphrase: vyitm684
    * Will need to be inputted any time you need to interact with your CA
* PEM Passphrase: vyitm684
    * Was asked for this but am not sure what it is
* Common Name: pressed *Enter* to acccept the default name

***

## [3. OpenServer Setup](https://www.digitalocean.com/community/tutorials/how-to-set-up-and-configure-an-openvpn-server-on-ubuntu-20-04)

### 3.0 Repeat Step 1 (except for deactivating DigitalOcean)
* Deploy a new Droplet to server as the OpenVPN Server
    * Droplet IP address: 142.93.116.143
    * Default root password: ITM684_Vy
* Repeat Steps 1.1 to 1.5

### 3.1 Installing OpenVPN and Easy-RSA
* Note: A majority of these steps are the same as Step 2, except for installing openvpn in addition to easy-rsa and specifying the directory owner
* `sudo apt update`
* `sudo apt install openvpn easy-rsa`
* Press *Y* and *Enter*
* `mkdir ~/easy-rsa`
* `ln -s /usr/share/easy-rsa/* ~/easy-rsa/`
* `sudo chown vy ~/easy-rsa`
    * Ensure that the `easy-rsa` directory's owner is the non-root sudo user
* `chmod 700 ~/easy-rsa`

### 3.2 Creating a PKI for OpenVPN
* Note: A majority of these steps are the same as Step 2, except for the order of creating the vars file and initializing the PKI
* `cd ~/easy-rsa`
* `nano vars`
* Paste the following lines into the file:
    ```
    set_var EASYRSA_ALGO "ec"
    set_var EASYRSA_DIGEST "sha512"
    ```
* Press *CTRL+X* to exit
* Press *Y* and *Enter*
* `./easyrsa init-pki`

### 3.3 Creating an OpenVPN Server Certificate Request and Private Key
* If not already in `easy-rsa` directory
    * `cd ~/easy-rsa`
* `./easyrsa gen-req server nopass`
    * Call the `easyrsa` with the `gen-req` option followed by a Common Name for the machine 
    * To follow the tutorial, the CN will be `server`
    * The `nopass` option will make it so the request file is not password-protected
    * Ran into the same error as Step 2:
    * Solve error by making a copy of the vars folder outside of the pki, naming it vars, then removing the old vars folder
        ```
        cp -r vars vars1
        rm -r vars
        ```
* Press *Enter* to accept the CN `server`
* A private key for the server and a certificate request file called `server.req` was created
* If still in `easy-rsa` directory
    * `cd ~`
* `sudo cp /home/vy/easy-rsa/pki/private/server.key /etc/openvpn/server/`
    * To copy the server key 

### 3.4 Signing the OpenVPN Server's Certificate Request
* `scp /home/vy/easy-rsa/pki/reqs/server.req vy@138.68.93.182:/tmp`
    * Use the transfer method, scp, to copu the server.req certificate request to the CA server
    * **IF YOU CHANGED THE FIREWALL CONFIG LIKE IN THE PIC, THE COMMAND IS DIFFERENT. CHECK PHOTOS**

* Type *yes* and press *Enter*
* Enter regular user's password

Now, log in to the CA server as the non-root user created in Step 1
* `ssh vy@138.68.93.182`
* Enter the regular user's password
* `cd ~/easy-rsa`
* `./easyrsa import-req /tmp/server.req server`
    * Import the certificate request from the OpenVPN server
* `./easyrsa sign-req server server`
    * Sign the request by running the `easyrsa` script with `sign-req` option, the request type (can be `client` or `server`) and the CN
* Type *yes* and press *Enter*
* Type the CA private key password (vyitm684) and press *Enter*
* Now, the `server.crt` file contains the OpenVPN server's public encryption key as well as a signature from the CA server
    * The signature tells anyone who trusts the CA server that they can also trust the OpenVPN server
* `scp pki/issued/server.crt vy@142.93.116.143:/tmp`
    * **IF YOU CHANGED THE FIREWALL CONFIG LIKE IN THE PIC, THE COMMAND IS DIFFERENT. HAVE TO ADD THE PORT NUMBER**
* Type *yes* and press *Enter*
* Enter the regular user's password
* `scp pki/ca.crt vy@142.93.116.143:/tmp`
    * **IF YOU CHANGED THE FIREWALL CONFIG LIKE IN THE PIC, THE COMMAND IS DIFFERENT. HAVE TO ADD THE PORT NUMBER**
* Enter the regular user's password

Go back to the OpenVPN server
* `sudo cp /tmp/{server.crt,ca.crt} /etc/openvpn/server`
* Enter the regular user's password

### 3.5 Configuring OpenVPN Cryptographic materian
* Add an extra shared secret key that the server and all clients will use
    * Used by OpenVPN server to perform quick checks on incoming packets. If packet is signed using pre-shared key, the server will process it. If not, then the server will discard it
* `cd ~/easy-rsa`
* `openvpn --genkey --secret ta.key`
    * A `ta.key` file is created
* `sudo cp ta.key /etc/openvpn/server`
    * Copy the file 
* Enter the regular user's password

### 3.6 Generating a Client Certificate and Key Pair
* Go to home directory if not in it
    * `cd ~`
* `mkdir -p ~/client-configs/keys`
    * Generate a single client key and certificate pair
* `chmod -R 700 ~/client-configs`
    * Lock down the directory's permissions as a security measure
* `cd ~/easy-rsa`
* `./easyrsa gen-req client1 nopass`
    *  Certificate/key pair CN: client1 (following the tutorial)
* Press *Enter* to confirm the CN
* `cp pki/private/client1.key ~/client-configs/keys/`
    * Copy the certificate/key pair into the the directory created at the beginning of 3.6
* `scp pki/reqs/client1.req vy@138.68.93.182:/tmp`
    * Transfer the `client1.req` file to the CA server
    * **IF YOU CHANGED THE FIREWALL CONFIG LIKE IN THE PIC, THE COMMAND IS DIFFERENT. HAVE TO ADD THE PORT NUMBER**
* Enter the regular user's password

Go to the CA server
* **IF YOU CHANGED THE FIREWALL CONFIG LIKE IN THE PIC, THE COMMAND IS DIFFERENT TO SSH INTO THE CA SERVER**
    * `ssh vy@138.68.93.182 -21999`
* If not already in `easy-rsa` directory
    * `cd ~/easy-rsa`
* `./easyrsa import-req /tmp/client1/req client1`
    * Import the certificate request 
* `./easyrsa sign-req client client1`
    * Sign the request by running the `easyrsa` script with `sign-req` option, the request type, this time with the `client` request type and the `client1` CA name
* Type *yes* and press *Enter*
* Type the CA private key password (vyitm684) and press *Enter*
* Now, a `client1.crt` file is created
* `scp pki/issued/client1.crt vy@142.93.116.143:/tmp`
* Enter the regular user's password

Go back to the OpenVPN server
* Go to home directory `cd ~`
* `cp /tmp/client1.crt ~/client-configs/keys/`
    * Copy the client certificate 
* `cp ~/easy-rsa/ta.key ~/client-configs/keys/`
    * Copy the `ta.key` file
* `sudo cp /etc/openvpn/server/ca.crt ~/client-configs/keys/`
    * Copy the `ca.crt` file
* Enter the regular user's password
* `sudo chown vy.vy ~/client-configs/keys/*`
    * Set the permission for the sudo user

### 3.7 Configuring OpenVPN
* Note: use edited command instead of :
    * `sudo cp /usr/share/doc/openvpn/examples/sample-config-files/server.conf.gz /etc/openvpn/server/`
    * Ran into an error stating:
        ```
        cp: cannot stat '/home/vy/usr/share/doc/openvpn/examples/sample-config-files/server.conf.gz': No such file or directory
        ```
    * After performing ls, found that there is no `server.conf.gz` file, only `server.conf`
* Continue with edited command
* `sudo cp /usr/share/doc/openvpn/examples/sample-config-files/server.conf /etc/openvpn/server/`
    * Copy the sample `server.conf` file
* Because file was not zipped, the following command is not necessary
    * `sudo gunzip /etc/openvpn/server/server.conf.gz`
* `sudo nano /etc/openvpn/server/server.conf`
    * Open the new file for editing
* HMAC Section
    * Find the `tls-auth ta.key 0 # This file is secret` line
    * Add a `;` to the front of the line to comment it out
    * Add a new line after that line: `tls-crypt ta.key`
* Cryptographic Ciphers Section
    * Find the `cipher AES-256-CBC` line (scroll down)
    * Add a `;` to the front of the line
    * Add a new line after that line: `cipher AES-256-GCM`
    * Add a new line after that line: `auth SHA256`
* Diffie-Hellman Parameters 
    * Find the `dh dh2048.pem` line (scroll up)
    * Add a `;` to the front of the line
    * Add a new line after that line: `dh none`
* Daemon Privileges
    * Find and uncomment the `user nobody` line (delete the `;` before the line) (scroll down)
    * Rename `group nobody` to  `group nogroup` and uncomment it 
* Push DNS Changes to Redirect All Traffic Through the VPN
    * Find and uncomment the `push "redirect-gateway def1 bypass-dhcp"` line (scroll up)
    * Uncomment the 2 lines below it `push "dhcp-option DNS 208.67.222.222"` and `push "dhcp-option DNS 208.67.220.220"`
* Change the Default Port and Protocol for the OpenVPN Server
    * Find the line `port 1194` (scroll up near top of file)
    * Change `1194` to `443`
    * Find and uncomment `proto tcp`
    * Find and comment `proto udp`
    * Find the `explicit-exit-notify` line (at the end of file) and change the valeue to `0`
* Save and close the file
    * Press *CTRL+O* to save
    * Press *CTRL+X* to exit

### 3.8 Adjusting the OpenVPN Server Networking COnfiguration
* `sudo nano /etc/sysctl.conf`
    * Open the file
    * Note: I mistook the 'l' in sysctl as a '1' and spent 10 minutes resolving my issue
* Enter the regular user's password
* Add this line at the bottom of the file
    * `net.ipv4.ip_forward = 1`
* Save and close the file
* To read the file and load new values for the current session: `sudo sysctl -p`

### 3.9 Firewall Configuration
* To allow OpenVPN through the firewall, masquerading needs to be enabled
    * Masquerading is an iptables concept that provides quick dynamic network address (NAT) to correctly route client connections
* `ip route list default`
    * Find the public network interface
* Output: `default via 142.93.112.1 dev eth0 proto static`
* `sudo nano /etc/ufw/before.rules`
    * Open the file
* Towards the top of the file, after the `#ufw-before-forward` line, add the following lines:
    ```
    # START OPENVPN RULES
    # NAT table rules
    *nat
    :POSTROUTING ACCEPT [0:0]
    # Allow traffic from OpenVPN client to eth0 (change to the interface you discovered!)
    -A POSTROUTING -s 10.8.0.0/8 -o eth0 -j MASQUERADE
    COMMIT
    # END OPENVPN RULES
    ```
    * This will set the default policy for the `POSTROUTING` chain in the `nat` table and masquerade any traffic coming from the VPN
* Save and close the file
* `sudo nano /etc/default/ufw`
    * Open the file
* Find the `DEFAULT_FORWARD_POLICY` and change the value from `DROP` to `ACCEPT`
    * This will tell UFW to allow frowarded packets by default too
* Save and close the file
* `sudo ufw allow 443/tcp`
    * Because we changed the port number and protocol earlier, we have to adjust the firewall to allow TCP traffic to port 443, which is to the OpenVPN
* `sudo ufw allow OpenSSH`
* Disable and re-enable UFW to restart it and load the changes 
    * `sudo ufw disable`
    * `sudo ufw enable`

### 3.10 Starting OpenVPN
* Open the OpenVPN server and ssh into it if not done so already (`ssh vy@142.93.116.143`)

* In the Open VPN server
* Configure OpenVPN to start up at boot so you can connect to your VPN at any time as long as your server is running
    * `sudo systemctl -f enable openvpn-server@server.service`
* Enter the regular user's password
* Start the OpenVPN service
    * `sudo systemctl start openvpn-server@server.service`
* Check to see that the OpenVPN service is active
    * `sudo systemctl status openvpn-server@server.service`
* *CTRL+C* to exit

### 3.11 Creating the Client Configuration Infrastructure
* Create a directory to store client configuration files 
    * `mkdir -p ~/client-configs/files`
* Copy and example client configuration file into the directory to use as the base configuration
    * `cp /usr/share/doc/openvpn/examples/sample-config-files/client.conf ~/client-configs/base.conf`
* `nano ~/client-configs/base.conf`
* Change the Default Port and Protocol
    * Find the `remote my-server-1 1194` line 
    * **THE COMMAND IS DIFFERENT**
    * Change the port number to 443 and add in IP address of your server
        * So, the line is `remote 142.93.116.143 443`
    * Find the and uncomment the `;proto tcp` line (scroll up)
    * Comment the `proto udp` line
* Daemon Privileges
    * Find and uncomment the `user nobody` line (delete the `;` before the line) (scroll down)
    * Rename `group nobody` to  `group nogroup` and uncomment it 
* Find the `ca ca.crt` line
* Comment it and the 2 lines that follow it 
* HMAC Section
    * Find and comment the `tls-auth ta.key 1` line (scroll down)
* Cryptographic Ciphers Section
    * Find the `cipher AES-256-CBC` line
    * Add a `;` to the front of the line
    * Add a new line after that line: `cipher AES-256-GCM`
    * Add a new line after that line: `auth SHA256`
* Anywhere in the file, add `key-direction 1`
* Add commented out lines to handle methods that Linux based VPN clients will use for DNS resolution
    * For client that do not use systemd-resolved to manage DNS  (clients who rely on the resolvconf utility to update DNS information)
        ```
        ; script-security 2
        ; up /etc/openvpn/update-resolv-conf
        ; down /etc/openvpn/update-resolv-conf
        ```
    * For clients that use systemd-resolved for DNS resolution
        ```
        ; script-security 2
        ; up /etc/openvpn/update-systemd-resolved
        ; down /etc/openvpn/update-systemd-resolved
        ; down-pre
        ; dhcp-option DOMAIN-ROUTE .
        ```
* Save and close the file
* Create a script to compile your base configuration
    * `nano ~/client-configs/make_config.sh`
* Add the following content:
    ```
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
        <(echo -e '</key>\n<tls-crypt>') \
        ${KEY_DIR}/ta.key \
        <(echo -e '</tls-crypt>') \
        > ${OUTPUT_DIR}/${1}.ovpn
    ```
* Save and close the file
* Mark this file as configurable
    * `chmod 700 ~/client-configs/make_config.sh`
    * Rather than having to manage the client's configuration, certificate, and key files separately, all the required information is stored in one place

### 3.12 Generating Client Configurations
* `cd ~/client-configs`
* `./make_config.sh client1`
    * To check that the command ran:
        * `ls ~/client-configs/files`
        * The output should be `client1.ovpn`
* Go to local computer terminal
* Paste the following command to copy the `client1.ovpn` file to the home directory
    * `sftp vy@142.93.116.143:client-configs/files/client1.ovpn ~/`
* Go to Finder
* Search for `client1.ovpn` to confirm that the file was successful copied
    
### 3.13 Installing the Client Configuration
* Install Tunnelblick from [here](https://tunnelblick.net/downloads.html)
* Follow the install prompts
* Towards the end, select *I have configuration files*
* In Finder, find the `client1.ovpn` file and drag it to the Tunnelblick icon on the top menu bar
* Click on the Tunnelblick icon
* Select *Connect client1*

### 3.14 Testing Your VPN Connection
* Open a browser and go to [DNSLeakTest](https://tunnelblick.net/downloads.html)
* Disconnect from the VPN connection
* You should see the IP address assigned by your ISP and as you appear to the world
    * Mines is 76.173.105.69 from Honolulu, United States
* Click on Extended Test to see which DNS servers you are using
    * Mines is 98.145.34.141
* Then, go back to the previous page
* Connect to the VPN
* Refresh the DNSLeakTest page and you should see a different IP address (I do not)


when you're donee, power down the servers, take snapshots, then power up the server and start with the algovpn portion
AlgoVPN
* ssh into your vpn server
* geta  copy of algo and install algo's core dependencies
    * git clone https://github/trailofbits/algo.git
    * sudo apt install -y --no=install-recommends python3-virtualenv
* cd algo 
    * you have to be in the algo directory to run the next commands
* editing the config file
    * change qireguard port to 53
    * change ssh port to 21999
    * add/edits names of users for wireguard and ipsec
* while still in the algo directory
    * sudo /algo
....
* at some point, it'll ask you for the ip address of your server, provide the ip address of your digital ocean vpn server
* if it goes well, you get a green response/success msg
* if it doesn't go well, 


