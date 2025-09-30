# pfSense_firewall_home_lab

## Introduction
This lab provides a comprehensive guide for setting up a network security lab focused on firewall configuration and attack mitigation. The lab consists of a virtual environment featuring an external attacker machine (**Kali Linux**) and a victim host (**Ubuntu Desktop**), protected by the **pfSense** open-source firewall. The primary objective is to simulate a Denial of Service (DoS) attack using hping3 and analyze the firewall's capability to detect, block, and log the malicious traffic.

## Objective
The objective of this lab is to:
* Set up a pfSense firewall with separate WAN (Bridged) and LAN (Internal) segments.
* Configure static routing on Kali Linux to reach the internal Ubuntu Desktop victim.
* Demonstrate a DoS attack using hping3 and analyze the packet flow in Wireshark.
* Mitigate the attack using a stateful pfSense firewall rule on the WAN interface.
* Monitor and analyze pfSense logs to verify the blocking mechanism.
* Develop practical defensive and network configuration skills applicable to securing network infrastructure.

## Requirements and Tools
| Category | Requirements |
| :--- | :--- |
| **Operating Systems** | pfSense, kali-linux, ubuntu |
| **System Requirements** | RAM: $\ge$ 8 GB, Disk Space: 50 GB free |
| **Security Tools** | hping3, Wireshark, pfSense Firewall |
| **Virtualization Software** | VirtualBox |

## Step 1: Setting Up Virtual Machines

### 1.1 Create pfSense VM
* OS Type: FreeBSD 64-bit
* Resources: 2 vCPU / 2 GB RAM / 20 GB disk.
* Adapter 1 (WAN): **Bridged Adapter**
* Adapter 2 (LAN): **Internal Network**, name: LabNet
* Install pfSense (accept defaults).

### 1.2 Create Kali VM
* OS Type: Debian 64-bit
* Resources: 2 vCPU / 2 GB RAM / 15 GB disk.
* Adapter 1: **Bridged Adapter**
* Install Kali Linux.

### 1.3 Create Ubuntu VM
* OS Type: Ubuntu 64-bit
* Resources: 2 vCPU / 2 GB RAM / 15 GB disk.
* Adapter 1: **Internal Network**, name: LabNet
* Install Ubuntu Desktop.

## Step 2: pfSense Initial Configuration and Interface Setup

### 2.1 Interface Assignment and WAN Access
1.  Boot pfSense and access the console menu.

<img src='https://github.com/TanunM/pfSense_firewall_home_lab/blob/main/gallery/pfsense_terminal.png'/>

2.  Via the console, enter the shell (option 8) and temporarily disable the firewall to gain WebGUI access:
    ```bash
    pfsense -d
    ```
3.  Access the WebGUI of pfSense using the WAN IP using default credentials (**admin / pfsense**).

<img src='https://github.com/TanunM/pfSense_firewall_home_lab/blob/main/gallery/pfsense_login.png'/>

<img src='https://github.com/TanunM/pfSense_firewall_home_lab/blob/main/gallery/pfsense_uncheck_defualt.png'/>

4.  Navigate to **Firewall** -> **Rules** -> **WAN** -> **Add** and create a rule to allow GUI access from the home network:
    * Action: Pass, Protocol: TCP, Source: WebGUI IP.
    * Destination: pfsense WAN, Port: 443.
    * Apply and save the rule.

<img src='https://github.com/TanunM/pfSense_firewall_home_lab/blob/main/gallery/GUI_wan_rule.png'/>

<img src='https://github.com/TanunM/pfSense_firewall_home_lab/blob/main/gallery/GUI_ip.png'/>

### 2.2 Configure LAN DHCP
1.  Navigate to **Services** -> **DHCP Server** -> **LAN**.
2.  Tick **Enable DHCP server on LAN interface**
3.  Range: 192.168.1.100 – 192.168.1.199
4.  DNS servers: home router IP
5.  Save and Apply Changes.

<img src='https://github.com/TanunM/pfSense_firewall_home_lab/blob/main/gallery/DHCP_server.png'/>

### 2.3 Initial WAN Access Rule pfSense
Add a rule to the **WAN** interface to allow traffic from Kali to the Ubuntu victim:
* Navigate to **Firewall** -> **Rules** -> **WAN** -> **Add**
* Action: Pass, Protocol: ICMP
* Source: Kali Linux IP
* Destination: pfSense IP

<img src='https://github.com/TanunM/pfSense_firewall_home_lab/blob/main/gallery/kali_pfsense_ICMP.png'/>

### 2.4 Initial WAN Access Rule Ubuntu
Add a rule to the **WAN** interface to allow traffic from Kali to the Ubuntu victim:
* Navigate to **Firewall** -> **Rules** -> **WAN** -> **Add**
* Action: Pass, Protocol: Any
* Source: Kali Linux IP
* Destination: Ubuntu LAN IP

<img src='https://github.com/TanunM/pfSense_firewall_home_lab/blob/main/gallery/allow_TCP_ubuntu.png'/>

<img src='https://github.com/TanunM/pfSense_firewall_home_lab/blob/main/gallery/kali_pfsense_ping.png'/>

## Step 3: Routing and Connectivity Setup

### 3.1 Kali Linux Routing
On the Kali Linux terminal, add a static route to provide reachability to the internal **192.168.1.0/24** subnet, routing through the pfSense IP:
```bash
sudo ip route add 192.168.1.0/24 via <PFSENSE_IP>
```
### 3.2 Ubuntu Connectivity Verification
1. On the Ubuntu VM, verify the DHCP-assigned IP address.
2. Test outbound Internet reachability through pfSense.
```bash
ifconfig
ping -c3 google.com
```
3. Ping from Kali Linux to the Ubuntu machine's IP to confirm the initial firewall rule and static route are working.

## Step 4: Wireshark Attack Analysis

1. On the Ubuntu, install Wireshark:
```bash
sudo apt install -y wireshark
```
2. Start the Wireshark
```bash
sudo wireshark
```

## Step 5: Simulate DoS Attack
1. Launching the Flood from Kali:
With Wireshark running on Ubuntu, execute the DoS flood command on the Kali, targeting the Ubuntu IP
```bash
sudo hping3 --flood -S -p 80 192.168.1.100
```
2. Attack Monitoring:
View the Packets spike in Wireshark on the Ubuntu, confirming the DoS is successfully.

<img src='https://github.com/TanunM/pfSense_firewall_home_lab/blob/main/gallery/wireshark_DoS.png'/>

## Step 6: Firewall rule and Log Analysis
1. Implementing the Block Rule:
Navigate to the pfSense WebGUI and add a new rule to block the malicious traffic. This rule must be added as the TOP rule on the WAN interface to overrule "Allow" rules.
  * Navigate to Firewall → Rules → WAN → Add (TOP).
  * Action: Block, Protocol: ICMP
  * Source: Kali-IP
  * Destination: 192.168.1.100
  * Log: check the box
  * Save and apply changes

<img src='https://github.com/TanunM/pfSense_firewall_home_lab/blob/main/gallery/kali_ubuntu_TCP.png'/>

2. Verification and Log Check:
  * You will see that the traffic halts in Wireshark on the Ubuntu machine.
  * On the pfSense WebGUI, navigate to Status → System Logs → Firewall and view the logs.
  * You will see that the malicious flood traffic attempts are being logged and blocked by the new rule.

<img src='https://github.com/TanunM/pfSense_firewall_home_lab/blob/main/gallery/Dos_blocked.png'/>

<img src='https://github.com/TanunM/pfSense_firewall_home_lab/blob/main/gallery/DoS_blocked_pfsense.png'/>

## Troubleshooting
* Kali cannot ping Ubuntu: Check Static Route On the Kali, verify the static route is correctly added using the pfSense WAN IP as the gateway
* DoS attack block rule doesn't work: Rule Order and Action: In pfSense Firewall, ensure the Block Kali DoS rule is placed above the initial "Allow Kali access to Ubuntu" rule.
* 
## Key Lab Learnings
* Network Configuration: Gained experience with Bridged vs. Internal Networking, DHCP configuration, and static routing.
* Firewall Administration: Learned to install pfSense, assign interfaces, manage WAN access, and configure stateful firewall rules with logging.
* Attack Simulation: Understood how to use hping3 to simulate common DoS floods (ICMP/SYN).
* Forensic Analysis: Used Wireshark to monitor network traffic spikes and confirmed the effectiveness of the security policy via pfSense firewall logs.

## Reference
* [PFSENSE Firewall Home Lab](https://youtu.be/-yRvfbElT7M)
* [Deploying a Virtual pfSense Firewall](https://www.youtube.com/watch?v=BcdJbBAnAKo)
