# Custom Cloud VPN Server Project: Installation Documentation

<details>
<summary><font size="4">Step 1: Inial server setup</font></summary>

* Deactivate DigitalOcean account and create a new one
* Deploy a new Droplet to server as the Certificate Authority (CA) Server
    * CA Server IP address: 159.223.133.122
    * Default root password: ITM684.Vy
* Log in as root if not already
    * `ssh root@24.199.92.229`
* Create a new user and grant privileges
    * `adduser vy`
    * `usermod -aG sudo vy`
</details>


<details>
<summary><font size="4"><a href="https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-20-04">Step 2: Set up a basic firewall</a></font></summary>

* `ufw app list`
    * The UFW firewall will make sure only connections to certain service are allowed
* `ufw allow OpenSSH`
    * To make sure the firewall allows SSH connections 
* `ufw enable`
* `su - vy`
***
### Change the default ssh port to 41235
* `sudo ufw allow 41235`
* `sudo nano /etc/ssh/sshd_config`
* Search for `#Port 22` line
* Remove the `#` and change the port number to `41235`
* *Ctrl+X* to save and exit
* `sudo systemctl restart ssh`
* Ran into an error:
    * Command `ss -an | grep 41235` to verify that ssh is listening was not outputing anything
    * Then, the console was logged out and an *SSH Connection Lost* error message appeared
* Solved by:
    * Going to local machine's terminal
    * `ssh vy@24.199.92.229`
    * `sudo apt-get update`
    * `sudo apt-get install openssh-server`
    * `sudo systemctl start ssh`
        * To start the ssh service again
* Then re-ran the command `ss -an | grep 41235`
    * Verify that ssh is listening
* `exit`
* ssh back in using the command
    * `ssh vy@24.199.92.229 -p41235`
* `ufw status`
    * To see if SSH connections are still allowed
    * Currently, the firewall is blocking all connections except for SSH
* 

from own terminal, ssh into DO server
</details>

<details>
<summary><font size="4"><a href="https://www.digitalocean.com/community/tutorials/how-to-set-up-and-configure-a-certificate-authority-ca-on-ubuntu-20-04">Step 3: Public Key Infrastructure setup</a></font></summary>

* easy-rsa: a CA management tool used to generate a private key and public root certificate 
    * The public root certificate is used to sign requests from clients and servers
* Note: be logged in as the non-root sudo user
* `sudo apt update`
* `sudo apt install easy-rsa`
* `mkdir ~/easy-rsa`
    * Note: DO NOT use sudo going forward because the normal user should manage and inteact with the CA without needing elevated privileges
* Make a symbolic link so that updates to the easy-rsa is automatically reflected
    * `ln -s /usr/share/easy-rsa/* ~/easy-rsa/`
* `chmod 700 /home/vy/easy-rsa`
* Initialize the PKI inside the easy-rsa directory
    * `cd ~/easy-rsa`
    * `./easyrsa init-pki`
</details>

<details>
<summary><font size="4">Step 4: CA setup</font></summary>

* `nano vars`
    * Paste the following into the file:
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
* Save and exit
* `./easyrsa build-ca`
* Ran into an error:
    * vars folder duplicated in easy-rsa directory as well as in pki directory
* Solved by:
    * `mv vars vars1`
    * Renamed duplicated vars file in easy-rsa and kept vars file in pki
* Reran command `./easyrsa build-ca`
* Command Name: pressed *Enter* to accept default name
</details>

<details>
<summary><font size="4">Step 5: OpenVPN server setup</font></summary>

* `exit`
* Repeat steps 1 and 2 (except for deactivation of DigitalOcean account)
* OpenVPN Server IP address: 159.223.133.122
* Default root password: ITM684.Vy
* Log in as root if not already
    * `ssh root@165.227.87.242`
* Create a new user and grant privileges
    * `adduser vy`
    * `usermod -aG sudo vy`
* Note: ran into similar error as before, resolved the same way
</details>

<details>
<summary><font size="4"><a href="https://www.digitalocean.com/community/tutorials/how-to-set-up-and-configure-an-openvpn-server-on-ubuntu-20-04">Step 6: OpenVPN and Easy-RSA setup</a></font></summary>

* Log in as non-root user on DO OpenVPN Server
    * `ssh vy@165.227.87.242 -p41235`
* `sudo apt update`
* `sudo apt install openvpn easy-rsa`
* `mkdir ~/easy-rsa`
* `ln -s /usr/share/easy-rsa/* ~/easy-rsa/`
* `sudo chown vy ~/easy-rsa`
* `sudo chown 700 ~/easy-rsa`
</details>

<details>
<summary><font size="4">Step 7: PKI for OpenVPN</font></summary>

* `cd ~/easy-rsa`
* `nano vars`
* Paste the following lines into the file:
    ```
    set_var EASYRSA_ALGO "ec"
    set_var EASYRSA_DIGEST "sha512"
    ```
* Save and exit
* `./easyrsa init-pki`
</details>

<details>
<summary><font size="4">Step 8: OpenVPN Server Certificate Request and Private Key</font></summary>

* `./easyrsa gen-req server nopass`
    * Call the `easyrsa` with the `gen-req` option followed by a Common Name for the machine 
    * To follow the tutorial, the CN will be `server`
    * The `nopass` option will make it so the request file is not password-protected
* Ran into the same error as Step 4:
    * vars folder duplicated in easy-rsa directory as well as in pki directory
* Solved by:
    * `mv vars vars1`
    * Renamed duplicated vars file in easy-rsa and kept vars file in pki
* Reran command `./easyrsa gen-req server nopass`
* Press *Enter* to accept the CN `server`
* A private key for the server and a certificate request file called `server.req` was created
* `cd ~`
* Copy the server key
    * `sudo cp /home/vy/easy-rsa/pki/private/server.key /etc/openvpn/server/`
</details>

<details>
<summary><font size="4">Step 9: Singing the OpenVPN Server's Certificate Request</font></summary>

* The CA Server needs to know about the OpenVPN Server's Certificate Request and validate it
* Use `scp` to copy the request to the CA for signing
    * `scp -P 41235 ~/easy-rsa/pki/reqs/server.req vy@24.199.92.229:/tmp`
***
* SSH into CA Server `ssh vy@24.199.92.229 -p41235`
* `cd ~/easy-rsa`
* Import the Certificate Request from the OpenVPN Server
    * `./easyrsa import-req /tmp/server.req server`
* `./easyrsa sign-req server server`
    * Sign the request by running the `easyrsa` script with `sign-req` option, the request type (can be `client` or `server`) followed by the CN
* Now, the `server.crt` file contains the OpenVPN server's public encryption key as well as a signature from the CA server
    * The signature tells anyone who trusts the CA server that they can also trust the OpenVPN server
* Copy the `server.crt` and `ca.crt` files from the CA Server to the OpenVPN Server
    * `scp -P 41235 pki/issued/server.crt vy@165.227.87.242:/tmp`
    * `scp -P 41235 pki/ca.crt vy@165.227.87.242:/tmp`
***
* SSH into the OpenVPN Server `ssh vy@165.227.87.242 -p41235`
* `sudo cp /tmp/{server.crt,ca.crt} /etc/openvpn/server`
</details>

<details>
<summary><font size="4">Step 10: Configuring OpenVPN Cryptographic Material</font></summary>

* Add an extra shared secret key that the server and all clients will use
* `cd ~/easy-rsa`
* `openvpn --genkey --secret ta.key`
    * A `ta.key` file is created
* `sudo cp ta.key /etc/openvpn/server`
</details>

<details>
<summary><font size="4">Step 11: Generate a Client Certificate and Key Pair</font></summary>

* Generate a single client key and cerficate pair to create a script that will automatically generate client configuration files containing all of the required keys and certificates
* `cd ~`
* `mkdir -p ~/client-configs/keys`
* `chmod -R 700 ~/client-configs`
* `cd ~/easy-rsa`
* `./easyrsa gen-req client1 nopass`
* Press *Enter* to confirm the default CN (`client1`)
* `cp pki/private/client1.key ~/client-configs/keys/`
    * Copy the certificate/key pair into the client-configs directory
* Transfer the `client1.req` file to the CA server
    * `scp -P 41235 pki/reqs/client1.req vy@24.199.92.229:/tmp`
***
* SSH into CA Server `ssh vy@24.199.92.229 -p41235`
* `cd ~/easy-rsa`
* Import the Certificate Request from the OpenVPN Server
    * `./easyrsa import-req /tmp/client1.req client1`
* `./easyrsa sign-req client client1`
    * Sign the request by running the `easyrsa` script with `sign-req` option, but this time, with the `client` request type followed by the CN 
* Now a client certificate file named `client1.crt` is created
* Transfer this file back to the OpenVPN Server
    * `scp -P 41235 pki/issued/client1.crt vy@165.227.87.242:/tmp`
***
* SSH into the OpenVPN Server `ssh vy@165.227.87.242 -p41235`
* Copy the client certificate
    * `cp /tmp/client1.crt ~/client-configs/keys/`
* Copy the `ca.crt` and `ta.key` files
    * `cp ~/easy-rsa/ta.key ~/client-configs/keys/`
    * `sudo cp /etc/openvpn/server/ca.crt ~/client-configs/keys/`
* Now, the server and client cetificates and keys are generated and stored in the OpenVPN Server
</details>

<details>
<summary><font size="4">Step 12: Configuring OpenVPN</font></summary>

* Copy a smaple configuration file included in this software's documentation
    * `sudo cp /usr/share/doc/openvpn/examples/sample-config-files/server.conf /etc/openvpn/server/`
* `sudo nano /etc/openvpn/server/server.conf`
*  Change the Default Port and Protocol for the OpenVPN Server
    * Find the line `port 1194` and change to `443`
    * Find and uncomment `;proto tcp`
    * Find and comment `proto udp`
    * Find the `explicit-exit-notify` line (at the end of file) and change the value to `0`
* Diffie-Hellman Parameters 
    * Find and comment `dh dh2048.pem`
    * Add a new line after that line: `dh none`
* Push DNS Changes to Redirect All Traffic Through the VPN
    * Find and uncomment `;push "redirect-gateway def1 bypass-dhcp"` 
    * Uncomment the 2 lines below it `;push "dhcp-option DNS 208.67.222.222"` and `;push "dhcp-option DNS 208.67.220.220"`
* HMAC 
    * Find and comment `tls-auth ta.key 0 # This file is secret`
    * Add a new line after that line: `tls-crypt ta.key`
* Cryptographic Ciphers 
    * Find and comment `cipher AES-256-CBC`
    * Add a new line after that line: `cipher AES-256-GCM`
    * Add a new line after that line: `auth SHA256`
* Daemon Privileges
    * Find and uncomment `;user nobody`
    * Uncomment `;group nobody` and rename to `group nogroup`
* Save and exit
</details>

<details>
<summary><font size="4">Step 13: Adjusting the OpenVPN Server Network Configuration</font></summary>

* `sudo nano /etc/sysctl.conf`
* Add this line at the bottom of the file
    * `net.ipv4.ip_forward = 1`
* Save and exit
* To read the file and load new values for the current session: `sudo sysctl -p`
* Output shoud say: *net.ipv4.ip_forward = 1*
</details>

<details>
<summary><font size="4">Step 14: Firewall Configuration</font></summary>

* To allow OpenVPN through the firewall, masquerading needs to be enabled
    * Masquerading is an iptables concept that provides quick dynamic network address (NAT) to correctly route client connections
* Find the public network interface
    * `ip route list default`
* Look at and note the output and the interface:
    * *default via 165.227.80.1 dev eth0 proto static* 
    * Interface: *eth0*
* `sudo nano /etc/ufw/before.rules`
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
    * If your interface is not eth0, replace it in the `-A POSROUTING` line
* Save and exit
* `sudo nano /etc/default/ufw`
* Find `DEFAULT_FORWARD_POLICY` and change the value from `DROP` to `ACCEPT`
    * This will tell ufw to allow forwarded packets by default too
* Save and exit
* `sudo ufw allow 443/tcp`
    * * Because we changed the port number and protocol earlier, we have to adjust the firewall to allow TCP traffic to port 443
* `sudo ufw allow OpenSSH`
* Disable and re-enable UFW to restart it and load the changes 
    * `sudo ufw disable`
    * `sudo ufw enable`
</details>

<details>
<summary><font size="4">Step 15: Starting OpenVPN</font></summary>

* Configure OpenVPN to start up at boot so you can connect to your VPN at any time as long as your server is running
    * `sudo systemctl -f enable openvpn-server@server.service`
* Start the OpenVPN service
    * `sudo systemctl start openvpn-server@server.service`
* Check to see that the OpenVPN service is active
    * `sudo systemctl status openvpn-server@server.service`
* *CTRL+C* to exit
</details>

<details>
<summary><font size="4">Step 16: Creating the Client Configuration Structure</font></summary>

* `mkdir -p ~/client-configs/files`
* Copy and example client configuration file into the directory to use as the base configuration
    * `cp /usr/share/doc/openvpn/examples/sample-config-files/client.conf ~/client-configs/base.conf`
* `nano ~/client-configs/base.conf`
* Change the Default Port and Protocol
    * Find the and uncomment  `;proto tcp` 
    * Find and comment `proto udp`
    * Find `remote my-server-1 1194`  
    * Change the port number to 443 and add in IP address of the OpenVPN Server
        * `remote 165.227.87.242 443`
* Daemon Privileges
    * Find and uncomment `;user nobody`
    * Uncomment `;group nobody` and rename to `group nogroup`
* Find the `ca ca.crt` line
* Comment it and the 2 lines that follow it 
* HMAC 
    * Find and comment `tls-auth ta.key 1`
* Cryptographic Ciphers 
    * Find and comment `cipher AES-256-CBC`
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
* Save and exit
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
* Save and exit
* `chmod 700 ~/client-configs/make_config.sh`
</details>

<details>
<summary><font size="4">Step 17: Generating Client Configuration</font></summary>

* `cd ~/client-configs`
* `./make_config.sh client1`
    * To check that the command ran:
        * `ls ~/client-configs/files`
        * The output should be `client1.ovpn`
* Go to FileZilla
* Select SFTP
* Host: `165.227.87.242`
* Username: `vy`
* Password: *non-root user password*
* Port: `41235`

* client-configs > files > client1.ovpn
* Copy `client1.ovpn` to local machine
</details>

<details>
<summary><font size="4">Step 18: Install TunnelBlick to Test VPN Connection</font></summary>

* Install Tunnelblick from [here](https://tunnelblick.net/downloads.html)
* Follow the install prompts
* Towards the end, select *I have configuration files*
* In Finder, find the `client1.ovpn` file and drag it to the Tunnelblick icon on the top menu bar
* Click on the Tunnelblick icon
* Select *Connect client1*
***
* Disconnect from server
* Go to [ipleak.net](https://ipleak.net/) before 
* Should see IP address assigned by ISP
* Connect to TunnelBlick
* Refresh ipleak.net
* Should now see IP address of selected data center for VPN
</details>

<details>
<summary><font size="4">Step 19: Install Wireguard and IPSec on the same server using AlgoVPN script</font></summary>

* Follow instructions from [trailofbits](https://github.com/trailofbits/algo)
* Change Wireguard port to `53`
* IPSec

</details>

<details>
<summary><font size="4">Step 20: Create cron jobs</font></summary>

* Follow instructions from [https://www.hostinger.com/tutorials/cron-job](https://www.hostinger.com/tutorials/cron-job)
* Create/Edit a crontab file: `crontab -e`
* See a list of active scheduled tasks: `crontab -l`
* Check for all updates and install them every 24 hours
    * `0 0 * * * apt-get update && apt-get upgrade -y`
    * Run command apt-get upgrade at mighnight (0 hrs and 0 mins) every day (* * * *)
* Send all failed login attempts to a file every hour
    * `0 * * * * grep 'Failed passwords' /var/log/auth.log > ~/faillog.txt`
    * Run grep command to search for Failed passwords lines in the auth.log file 
* Clear the faillog.txt file every day at midnight
    * `0 0 * * * > ~/faillog.txt`

</details>

<details>
<summary><font size="4">Step 21: Install a different shell and color code</font></summary>

* `sudo apt install zsh`
    * https://www.digitalocean.com/community/tutorials/how-to-install-z-shell-zsh-on-a-cloud-server
* `zsh` to switch shells
* `bash` to switch back to bash
* `ps -p $$` to display current shell name
***
* Colorizing a bash prompt: 
    * `nano ~/.bashrc`
    * Commented: `PS1="\[\e]0;${debian_chroot:+($debian_chroot)}\u@\h: \w\a\]$PS1"`
    * Pasted: `PS1='\[\033[1;36m\]\u@\h: \w\[\033[0m\]\$ '`
    * Save and exit
    * Reload console
    * Prompt should now be cyan and bolded

* Colorizing a zsh prompt:
    * `nano ~/.zshrc`
    * Pasted: `PS1='%F{cyan}%1m%~%f '`
    * Save and exit
    * `source ~/.zshrc`
    * Prompt should now be cyan and bolded
</details>

<details>
<summary><font size="4">Step 22: Creating aliases</font></summary>

* bash: 
    * `nano ~/.bashrc`
    * Insert the following lines:
        * `alias c='clear'`
        * `alias ls='ls --color=auto'`
        * `alias ..='cd ..'`
    * Save and exit
    * Reload console

* zsh:
    * `nano ~/.zshrc`
    * Insert the following lines:
        * `alias c='clear'`
        * `alias ls='ls --color=auto'`
        * `alias ..='cd ..'`
    * Save and exit
    * `source ~/.zshrc`
</details>

