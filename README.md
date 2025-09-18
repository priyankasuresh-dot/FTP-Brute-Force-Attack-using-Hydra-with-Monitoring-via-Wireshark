# FTP-Brute-Force-Attack-using-Hydra-with-Monitoring-via-Wireshark
This project demonstrates a brute-force attack on an FTP service using Hydra and shows how to detect and analyze the attack using Wireshark. It highlights both offensive (attacking) and defensive (monitoring &amp; mitigation) aspects of cybersecurity.

# Tools & Environment

Victim Machine (Cyber Lab - Linux): Runs FTP service (port 21).

Kali Linux: Attacker machine running Hydra.

Wireshark / tcpdump: Capture and analyze FTP traffic (FTP transmits credentials in plaintext by default).

PuTTY / FTP client (FileZilla, lftp): To verify discovered credentials.

iptables: Simple host-based mitigation to block attacker IPs.

# Quick Background

What is FTP?
File Transfer Protocol used to move files between client and server. By default it sends credentials and data in plaintext, which makes it a frequent target for credential-stealing attacks.

What is Hydra? 
THC-Hydra is a fast, flexible brute-force tool that automates trying username/password combinations across many protocols (including FTP).

# Project Objective

Show how an FTP brute-force attack is performed using Hydra, how to observe that attack with packet captures (Wireshark/tcpdump), and how to mitigate it using iptables.

# Step-by-Step Process
1. Prepare the environment

Ensure the victim Linux machine has an FTP server running (e.g., vsftpd or proftpd) and is reachable at an IP like 192.168.1.76.

Start Kali Linux as the attacker machine and ensure network connectivity between the attacker and victim.

2. Install Hydra (if needed)

Hydra is often available on Kali. If not:

git clone https://github.com/vanhauser-thc/thc-hydra
# Follow build/install steps from the repository (configure, make, make install)

3. Create credential lists

Prepare text files with candidate usernames and passwords (one per line):

user.txt — possible usernames

password.txt — possible passwords

4. Run the brute-force attack

From Kali run:

hydra -L user.txt -P password.txt 192.168.1.76 ftp


-L user.txt: username list

-P password.txt: password list

192.168.1.76: victim IP

ftp: protocol

Hydra will try combinations and report any valid credentials found.

5. Verify access with an FTP client

Use FileZilla, lftp or PuTTY (or ftp/curl) to connect to the victim with the discovered username/password and confirm file listing/upload/download works.

# Detection & Analysis (Wireshark / tcpdump)

Because FTP is plaintext, packet captures reveal authentication attempts:

Start a capture on the victim or a network tap:

sudo tcpdump -i eth0 -s 0 -w ftp_attack.pcap port 21

Or use Wireshark to capture live traffic and open the .pcap.

What to look for:

Repeated USER and PASS commands in short succession (many attempts from same source IP).

Repeated 530 Login incorrect responses for failures.

A 230 Login successful response when a credential pair succeeds.

Plaintext usernames and passwords visible in the packet payload.

Timing/frequency that indicate automated brute-force behavior (high rate of attempts).

Example Wireshark filters:

ftp — show FTP protocol traffic.

ftp.request.command == "USER" and ftp.request.command == "PASS" — to focus on auth exchanges.

Use Follow TCP Stream on a connection to view the full plaintext exchange.

# Simple Mitigation — Block Attacker IP (iptables)

Create a short script block_ftp.sh to block a malicious IP from reaching port 21:

#!/bin/bash
# Check for root
if [[ $EUID -ne 0 ]]; then
   echo "This script must be run as root."
   exit 1
fi

BLOCK_IP="$1"
if [ -z "$BLOCK_IP" ]; then
    echo "Usage: $0 <IP_ADDRESS>"
    exit 1
fi

# Block the IP for FTP (port 21)
iptables -A INPUT -s "$BLOCK_IP" -p tcp --dport 21 -j DROP
echo "Blocked FTP access from IP: $BLOCK_IP"


# Usage

chmod +x block_ftp.sh
sudo ./block_ftp.sh 10.55.255.145


# Verify

sudo iptables -L INPUT -v -n


Look for a rule showing DROP for tcp dpt:21 and the blocked source IP.

To block all incoming FTP:

sudo iptables -A INPUT -p tcp --dport 21 -j DROP

# Recommended Defensive Improvements (post-test)

Disable plaintext FTP; use SFTP (SSH) or FTPS (FTP over TLS).

Enforce strong passwords and account lockout (fail2ban or similar).

Rate-limit connections and use connection throttling.

Centralized logging and SIEM (e.g., deploy log collection and alerting) to detect repeated auth failures across hosts.

Use network segmentation so public-facing services are isolated.

# Summary

This project demonstrates how Hydra can be used to brute-force FTP credentials and how Wireshark/tcpdump can be used to observe the plaintext authentication exchanges and attack patterns. Quick mitigation via iptables blocks attacker IPs, but longer-term defenses—migrating off plaintext FTP, enforcing strong authentication, and automated blocking tools—are recommended.

