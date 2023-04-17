# Custom Cloud VPN Server Project: Installation Documentation

## Deploying an Ubuntu 22.04 LTS server to Digital Ocean

### [1. Initial Server Setup with Ubuntu 20.04](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-20-04)
* Granted administrative privileges to myself using `sudo usermod -aG bvytr`
    * Ran into an error:
        ```
        usermod: Permission denied.
        usermod: cannot lock /etc/passwd; try again later.
        ```
        * Resolved error by through [askubuntus](https://askubuntu.com/questions/991685/usermod-cannot-lock-etc-passwd); alternative command: `sudo usermod -aG audio bvytr` 
* Set up a basic firewall using the following commands:
    ```
    sudo ufw app list
    sudo allow OpenSSH
    sudo ufw enable
    sudo ufw status
    ```
    * The commands above will make it so the UFW firewall will block all connections except for SSH connections 
* Enabling external access for regular users
    * I realized that I had already enabled external access because I do not need to use a password to authenticate when I perform: `sudo command_to_run`

### [2. Setting Up and Configuring a Certificate Authority (CA)](https://www.digitalocean.com/community/tutorials/how-to-set-up-and-configure-a-certificate-authority-ca-on-ubuntu-20-04)
* Building a private CA will enable you to configure, test, and run programs that require encrypted connections between a client and a server
* Install `easy-rsa`: a CA management tool used to generate a private key and a public root certificate
    ```
    sudo apt update
    sudo apt install easy-rsa
    ```
* Prepare a Public Key Infrastructure (PKI) Directory
    * Create an `easy-rsa` directory: `mkdir ~/easy-rsa`
        * Note: do not use sudo
    * Create symbolic links pointing to the `easy-rsa` package files: `ln -s /usr/share/easy-rsa/* ~/easy-rsa/`
        * The package files are located in the `/usr/share/easy-rsa` folder
    * Restrict access to the PKI directory and ensure that only the owner can access it
        * `chmod 700 /home/bvytr/easy-rsa`
        * Permission 700 will give the owner rwx permissions while giving group and others no permissions
    * Initialize the PKI inside the easy-rsa directory
        ```
        cd ~/easy-rsa
        ./easyrsa init-pki
        ``` 
* Create a Certificate Authority
    * Create and populate `vars`
        ```
        cd ~/easy-rsa
        nano vars
        ```
    * Edit values in `vars`; do not leave any values blank
        * Note: Once you get to Step 3, you will only need 2 lines in the file (the last 2 lines)
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
    * Once finished, save close the file
        ```
        CTRL+X 
        Y 
        ENTER
        ```
    * Create the root public and private key pairs for your CA by running `./easyrsa build-ca nopass`
        * `build-ca` option generates the CA    
        * `nopass` will make it so you are not prompted for a password everytime you interact with your CA
        * Passphrase: (did not have because used the `nopass` option)
        * Common Name (CN; name used to refer to this machine): server
    * 2 important files:
        * `ca.crt`: CA's public certificate; what users, servers, and clients will use to verify
        * `ca.key`: private key that the CA uses to sign certificates for servers and clients
            * Should only be on your CA machine

### [3. Setting Up and Configuring an OpenVPN Server on Ubuntu 20.04](https://www.digitalocean.com/community/tutorials/how-to-set-up-and-configure-an-openvpn-server-on-ubuntu-20-04)
* Check to see you completed the prereqs:
    * An Ubuntu 20.04 server wiht a sudo non-root user and a firewall enabled (Step 1)
    * A separate Ubuntu 20.04 server set up as a private Certificate AUthority (CA) (Step 2; steps 1-3 of the CA set-up guide)
* Install OpenVPN and Easy-RSA
    * Update your OpenVPN Server's package index and install OpenVPN and Easy-RSA
        ```
        sudo apt update
        sudo apt install openvpn easy-rsa
        ```
    * Directory and link was already created in Step 2, so the following steps can be skipped:
        * Create a new directory on the OpenVPN Server as your non-root user: `mkdir ~/easy-rsa`  
        * Create a symlink from the `easyrsa` script into the `~/easy-rsa` directpry: `ln -s /usr/share/easy-rsa/* ~/easy-rsa/`
        * Ensure that the directory's owner is your non-root sudo user and restrict access to that user using `chmod`
            ```
            sudo chown bvytr ~/easy-rsa
            chmod 700 ~/easy-rsa
            ```
* Create a PKI for OpenVPN
    * Edit the vars file from what was done in Step 2
        * Open the file:
            ```
            cd ~/easy-rsa
            nano vars
            ```
        * Have the following lines in the file:
            ```
            set-var EASYRSA_ALGO        "ec"
            set_var EASYRSA_DIGEST      "sha512"
            ```
    * Initialize the PKI inside the easy-rsa directory
        ```
        cd ~/easy-rsa
        ./easyrsa init-pki
        ``` 
        * Type `yes` and submit
* Create an OpenVPN Server Certificate Request and Private Key
    * Generate a private key and Certificate Signing Request (CSR) 
    * If not already in the easy-rsa directory: `cd ~/easy-rsa`
    * Call the `easyrsa` with the `gen-req` option followed by a Common Name (CN) for the machine: 
        `./easyrsa gen-req server nopass`
        * This will create a private key for the server and a certiicate request called `server.req`
        * Note: Throughout the tutorial, OpenVPN Server's CN will be `server`
    * Copy the server key to the `./etc/openvpn/server` directory:
        `sudo cp /home/bvytr/easy-rsa/pki/private/server.key /etc/openvpn/server/`
* Signing the OpenVPN Server's Certificate Request
    * We created a Certificate Signing Request (CSR) in the previous step, so now the CA server needs to know about the `server` certificate and validate it 
    * Open the OpenVPN server as the non-root user copy the `server.req` certificate request to the CA:
        `scp /home/bvytr/easy-rsa/pki/reqs/server.req bvytr@192.168.64.3:/tmp
        scp /home/sammy/easy-rsa/pki/reqs/server.req sammy@your_ca_server_ip:/tmp
        
    

    

    