# SIEM & Threat Hunting With OSSEC

## Graphical Network Simulator-3 (GNS3):
GNS3 is used by hundreds of thousands of network engineers worldwide to emulate, configure, test and troubleshoot virtual and real networks. GNS3 allows you to run a small topology consisting of only a few devices on your laptop, to those that have many devices hosted on multiple servers or even hosted in the cloud.

## OSSEC Features:
- **Log based Intrusion Detection (LIDs)**: Actively monitors and analyzes data from multiple log data points in real-time

- **Rootkit and Malware Detection**: Process and file level analysis to detect malicious applications and rootkits

- **Active Response**: Respond to attacks and changes on the system in real time through multiple mechanisms including firewall policies, integration with 3rd parties such as CDN’s and support portals, as well as self-healing actions

- **Compliance Auditing**: Application and system level auditing for compliance with many common standards such as PCI-DSS, and CIS benchmarks

- **File Integrity Monitoring (FIM)**: For both files and windows registry settings in real time not only detects changes to the system, it also maintains a forensic copy of the data as it changes over time.

- **System Inventory**: Collects system information, such as installed software, hardware, utilization, network services, listeners and other information.
  
## Network Architecture & Configurations:
We simulate a small office network architecture which consists of 3 VLANs (Internal, Management, DMZ) and some typical networks devices (Router, Switch)
- **Router/Firewall**: Fortigate 7.0.14 (1 vCPU, 2GB RAM)
- **Switch**: CiscoIOSvL215 (1 vCPU, 768MB RAM)
- **OSSEC Server (Manager)**: Ubuntu 20.04 (2 vCPU, 2GB RAM)
- **Web Server**: Ubuntu 20.04 (2 vCPU, 2GB RAM)
- **File Server**: Ubuntu 20.04 (1 vCPU, 1GB RAM)
- **Staff Workstaion**: Windows 7 (2 vCPU, 2GB RAM)
- **VLAN 10 (Internal Network)**: This is the internal network which consists of Staff Workstation
- **VLAN 20 (Management)**: OSSEC Server is placed in seperated VLAN
- **VLAN 30 (DMZ)**: We simulate the DMZ of a small office network architect which consists of Web Server, File  Server

## Step-by-step deployment:
### Network Architecture in GNS3:
- Network Architecture: download and install necessary appliance from https://gns3.com/marketplace/appliances
- Fortigate:
  +  WAN interface (port 1) config:
  ```
    conf sys int
    edit port1
    set mode dhcp
    set allowaccess http https ping ssh
    set role wan
    end
  ```
  + We have finished configuring WAN interface of Fortigate. Now, we can access Fortigate Web Interface by using port1 IP Address
  +  LAN interface (port 2): Create 3 interfaces for 3 VLANs
  +  Configure each VLANs
  +  Configure Firewall Policy 
     +  VLAN1: allowed to access Internet and communicate with DMZ, VLAN2
     +  VLAN2: allowed to access Internet and communicate with DMZ, VLAN1
     +  VLAN3: allowed to access Internet (for packages installation), communicate with VLAN1, VLAN2
     +  Allow HTTP traffic from Internet to DMZ (forward traffic from WAN Interface to Web Server IP address)
- VLAN & Trunking Config (Switch CiscoIOSvL2):
  + Gi0/0 (connected to port 2 of Fortigate):
  ```
  enable
  conf t
  int Gi0/0
  switchport trunk encapsulation dot1q
  switchport mode trunk
  ```
  + Gi0/1 (VLAN 10):
  ```
  enable
  conf t
  int Gi0/1
  switchport mode access
  switchport access vlan 10
  ```
  + Gi0/2 (VLAN 20):
  ```
  enable
  conf t
  int Gi0/2
  switchport mode access
  switchport access vlan 20
  ```
  + Gi0/3 (VLAN 30 - Web Server):
  ```
  enable
  conf t
  int Gi0/3
  switchport mode access
  switchport access vlan 30
  ```
  + Gi1/0 (VLAN30 - File Server):
  ```
  enable
  conf t
  int Gi1/0
  switchport mode access
  switchport access vlan 30
  ```
- Note: Turn off NAT routing between VLAN in order for OSSEC Agent to easily connect to OSSEC Server without error
### Update system and install necessary packages:
- Update the system:
```
sudo apt update && sudo apt upgrade
```
- Install necessary packages:
```
sudo apt install -y php php-cli php-common libapache2-mod-php apache2-utils sendmail inotify-tools apache2 build-essential gcc make wget tar zlib1g-dev libpcre2-dev libpcre3-dev unzip libz-dev libssl-dev libpcre2-dev libevent-dev build-essential mysql-server git vim php7.2-gd php7.2-mysql
```
### OSSEC Server Installation:
- Enable & Start Apache
```
sudo systemctl enable apache2
sudo systemctl start apache2
sudo a2enmod rewrite
sudo systemctl restart apache2
```
- Run the following commands to install OSSEC Server (3.6.0) 
```
wget https://github.com/ossec/ossec-hids/archive/3.6.0.tar.gz
sudo tar -xvzf 3.6.0.tar.gz
sudo /home/hgadmin/ossec-hids-3.6.0/install.sh
```
- Or install the lastest using apt
```
wget -q -O - https://updates.atomicorp.com/installers/atomic | sudo bash

# Update apt data
sudo apt-get update

# Server
sudo apt-get install ossec-hids-server

# Agent
sudo apt-get install ossec-hids-agent
```
  + Then, provide our preferred input as prompted

OSSEC is located at ```/var/ossec```. To manage the agents and run specific OSSEC command: ```/var/ossec/bin``` 
  + Install Web User Interface
```
cd /tmp/
sudo git clone https://github.com/ossec/ossec-wui.git
sudo mv /tmp/ossec-wui /var/www/html
cd /var/www/html/ossec-wui
```
  + Set the permissions
```
sudo chown -R www-data:www-data /var/www/html/ossec-wui/
sudo chmod -R 755 /var/www/html/ossec-wui/

# Restart Apache and launch Web User Interface
sudo systemctl restart apache2
```
  + Run ```/var/ossec/bin/ossec_control start```

Now, we can access OSSEC Web UI at http://servers-ip/ossec-wui


### OSSEC Agent Installation:
- For Windows:
  + Download OSSEC Agent 3.6.0 for Windows at: https://updates.atomicorp.com/channels/atomic/windows/ossec-agent-win32-3.6.0-12032.exe
  + Download latest version at https://www.ossec.net/download-ossec/ (navigate to Windows tab)
  + Run setup wizard and follow the direction to complete the installation. 
- For Linux:
  + Follow the installation steps of OSSEC Server
  + At "provide our preferred input as prompted" step, we specify the kind of installation as agent (instead of server)
- Connect agent to server:
  + OSSEC Server:
    + Run ``` /var/ossec/bin/manage_agents```
    + Add agent
    + Extract key
  + OSSEC Agent (Windows):
    + Run OSSEC Agent Manager
    + Provide Server's IP and import key from OSSEC Server
    + Restart OSSEC Agent
    + Restart OSSEC Server ```/var/ossec/bin/ossec_control restart```
  + OSSEC Agent (Linux):
    + Run ```/var/ossec/bin/manage_agents```
    + Provide Server's IP and import key from OSSEC Server
    + Restart both OSSEC Agent and OSSEC Server ```/var/ossec/bin/ossec_control restart```

### Web Server:
In order to simualate a website with existing vulnerabilities, we install Damn Vulnerable Web Application (DVWA):
- Navigate to /var/www/html: ```cd /var/www/html```
- Clone DVWA repository ```git clone https://github.com/digininja/DVWA.git```
- Change directory name: ```mv DVWA/ dvwa/```
- Navigate to DVWA config directory: ```cd dvwa/config```
- Change config file name```mv config.inc.php.dist config.inc.php```
- Modify php.ini file: make sure **allow_url_include** and **allow_url_open** is setted to "On"
- Restart apache2: ```sudo systemctl restart apache2``` or ```sudo systemctl restart apache2.service```
- Modify database configuration of DVWA: ```cd /var/www/html/dvwa/config``` then using nano or vim to modify config.inc.php ```nano config.inc.php```
  - Change **db_user** to **root**
  - Change **db_password** to **""**
- Run ```mysql -u root```  then ```use mysql;```
- ```ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '';```
- Access DVWA at http://localhost/dvwa/setup.php
- Click **Create/Reset database** then **login**
Now, setup is completed. DVWA is ready.
### File Server:
Install WebDAV
- Enable WebDAV
```
sudo a2enmod dav
sudo a2enmod dav_fs
sudo systemcrl restart apache2.service
```
- Create sharing directory:
```
sudo mkdir /var/www/webdav/Documents
sudo mkdir /var/www/webdav/Downloads
```
Or, we can set up File Server by installing Samba
## References:
https://hendgrow.com/2020/10/01/ossec-open-source-hids-with-web-user-interface-updated-for-ubuntu-20-04-ossec-3-6-0/

https://thegioifirewall.com/dvwa-huong-dan-cai-dat-may-chu-dvwa-tren-ubuntu/

https://ossec-documentation.readthedocs.io/en/latest/manual/agentless.html

https://docs.fortinet.com/product/fortigate/7.0

https://docs.gns3.com/
