# Wireguard setup (Simplified)
[Wireguard](https://www.wireguard.com/)
1. [Install](https://www.wireguard.com/install/)
2. [Quickstart](https://www.wireguard.com/quickstart/)

---
Having read through the Install and Quickstart documentation, we can further make it easier to use in local systems without any additional external tools.

## Requirements
* 2 linux machines (The below steps have been tested on Fedora and RHEL)
* UDP port is open

## Configuration
The `wg genkey` command is used to generate the private key for each instance and the public key is obtained using `wg pubkey`

    wg genkey > private.key
    chmod 600 private.key
    wg pubkey < private.key

These keys will be used in the configuration file to be used by `wg-quick` command. A minimal configuration file needed to setup a working tunnel looks like below.   

On machine 1,

    # Server
    [Interface]
    ListenPort = 51820
    PrivateKey = QHe8oJwgPVaFLtOnye2u84DOpK4WzUadDRS27IyzoFQ=
    Address = 10.20.30.1/24

    [Peer]
    PublicKey = VYFNyv75Ift8l7SFzyqg/XAUOLTwjqtGdp/MWoRRbig=
    AllowedIPs = 10.20.30.0/30, 10.96.0.0/16
    Endpoint = 11.22.33.44:51820

On machine 2,

    # Client
    [Interface]
    ListenPort = 51820
    PrivateKey = ABOokrh3RAvXP3YLKU+BDUL/plNk/0dApwqegcjpNVY=
    Address = 10.20.30.2/24

    [Peer]
    PublicKey = 4qTrd1Q1SJPDzAoJUP6+cTkuw9FOKq62o6OSNDOOsmg=
    AllowedIPs = 10.20.30.0/30
    Endpoint = 44.33.22.11:51820

| Parameter  	| Description                                                                                                            	|
|------------	|------------------------------------------------------------------------------------------------------------------------	|
| ListenPort 	| UDP port used for communication. If this is not specified, a random port will be chosen                                	|
| Address    	| Tunnel endpoint IP address. This address is added to the wireguard interface                                           	|
| PrivateKey 	| Private key obtained using wg genkey                                                                                   	|
| PublicKey  	| "Peer" public key, obtained from the private key using wg pubkey                                                       	|
| AllowedIPs 	| Packets from these IP ranges will be allowed th pass through the wireguard interface, everything else will be dropped. 	|
| Endpoint   	| IP address of the server. Not to be confused with "Address" parameter                                                  	|

The configuration file could be stored anywhere or they could be stored in the `/etc/wireguard` folder from where the wg-quick command can load the configuration file based on the interface name.  

    Usage: wg-quick [ up | down | save | strip ] [ CONFIG_FILE | INTERFACE ]
    CONFIG_FILE is a configuration file, whose filename is the interface name
    followed by `.conf'. Otherwise, INTERFACE is an interface name, with
    configuration found at /etc/wireguard/INTERFACE.conf.

For example, we could name the configuration file as wg0, according to wireguard interface naming convention and store the configuration file in `/etc/wireguard/wg0.conf`

After this we can use wg-quick to quickly open and close a tunnel using the following commands

    wg-quick up wg0
    wg-quick down wg0

If the configuration file is stored somewhere else (not recommended since, the private key is stored in plaintext), for example in ~/.wireguard, then we would use the wg-quick as follows

    wg-quick up ~/.wireguard/wg0.conf
    wg-quick down ~/.wireguard/wg0.conf

It is important that the file name has the format, INTERFACE_NAME.conf, otherwise wg-quick will fail to create the interface.  

Before the tunnel can work, the port (ListenPort) needs to be open or the firewall disabled (not recommended).  
To open the port permanently (permanent is required for the automatically starting the tunnel after a restart, see next section)

    firewall-cmd --add-port 51280/udp 
    firewall-cmd --runtime-to-permanent

    # Or
    firewall-cmd --add-port 51280/udp --permanent
    firewall-cmd --reload

## Up/Down tunnel on system startup/shutdown
Login to root before creating the systemd.service (`man systemd.service`) file 

    sudo -i   
    # sudo -i, --login run login shell as the target user; default user is root    
    # Or  
    sudo su -l   
    # su -, -l, --login make the shell a login shell; default user is root  

Next create the service file

    cat > /usr/lib/systemd/system/wireguard.service <<EOF
    # /usr/lib/systemd/system/wireguard.service
    # See "wg-quick --help" for details

    [Unit]
    Description=Setup tunnel
    After=network.target multi-user.target

    [Service]
    Type=oneshot
    ExecStart=/bin/wg-quick up wg0
    ExecStop=/bin/wg-quick down wg0
    RemainAfterExit=yes
    
    [Install]
    WantedBy=multi-user.target
    EOF

After creating the service file, we have to create a link to the service for systemd to be able to use it. We can either link it manually, or use "systemctl enable" to install the link.

    # Still as root
    ln -s /usr/lib/systemd/system/wireguard.service /etc/systemd/system/wireguard.service
    systemctl start wireguard.service 

    # Alternatively
    systemctl enable wireguard.service
    systemctl start wireguard.service

    # Combined
    systemctl enable --now wireguard.service

Note: When using systemd to open/close the Wireguard tunnel, manually opening/closing the tunnel will conflict with the service.   
