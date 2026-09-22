# Host-Based Intrusion Prevention System (IPS)

A Host-Based Intrusion Prevention System (IPS) built using **Suricata**, **iptables**, and **NFQUEUE**.

The system runs Suricata directly on the victim machine and inspects incoming network traffic. When traffic matches a configured Suricata rule, Suricata can generate an alert and drop the packet before it reaches the victim system.

---

## Project Overview

The purpose of this project is to implement a practical host-based IPS that can:

* Monitor incoming network traffic
* Detect suspicious traffic using Suricata rules
* Inspect packets in real time
* Automatically drop packets that match blocking rules
* Log detected events
* Integrate with the Linux firewall through NFQUEUE

The current implementation uses ICMP traffic as a controlled laboratory test.

---

## Architecture

The project uses two virtual machines:

```text
┌──────────────────────┐
│     Attacker VM      │
│                      │
│  192.168.0.103       │
│                      │
│  Sends ICMP Traffic  │
└──────────┬───────────┘
           │
           │ ICMP
           ▼
┌──────────────────────────────┐
│          Victim VM           │
│                              │
│       192.168.0.107          │
│                              │
│        ┌───────────┐         │
│        │ iptables  │         │
│        └─────┬─────┘         │
│              │               │
│              ▼               │
│        ┌───────────┐         │
│        │ NFQUEUE 0 │         │
│        └─────┬─────┘         │
│              │               │
│              ▼               │
│        ┌───────────┐         │
│        │ Suricata  │         │
│        │    IPS    │         │
│        └─────┬─────┘         │
│              │               │
│         ┌────┴────┐          │
│         ▼         ▼          │
│      ACCEPT      DROP        │
│                  │           │
│                  X           │
└──────────────────────────────┘
```

> The IP addresses shown are from the test environment and may be different in another setup.

---

## Technologies Used

| Technology     | Purpose                                  |
| -------------- | ---------------------------------------- |
| Suricata 8.0.6 | Network traffic detection and prevention |
| iptables       | Linux firewall                           |
| NFQUEUE        | Sends packets to Suricata for inspection |
| Linux          | Host operating system                    |
| Kali Linux     | Virtual lab environment                  |

---

# Installation

## 1. Install Suricata

On the victim machine:

```bash
sudo apt update
sudo apt install suricata -y
```

Verify the installation:

```bash
suricata -V
```

Example:

```text
This is Suricata version 8.0.6 RELEASE
```

---

# Network Configuration

## 2. Find the Victim IP Address

On the victim:

```bash
ip -br addr
```

Check the routing table:

```bash
ip route
```

The network interface used in the current setup is:

```text
wlan0
```

---

## 3. Find the Attacker IP Address

On the attacker:

```bash
ip -br addr
```

In the test environment:

```text
Attacker: 192.168.0.103
Victim:   192.168.0.107
```

Test connectivity:

```bash
ping 192.168.0.107
```

At this stage, ICMP traffic should normally reach the victim.

---

# Suricata Rule Configuration

## 4. Check the Rules Directory

On the victim:

```bash
sudo ls -l /var/lib/suricata/rules/
```

The project uses:

```text
classification.config
suricata.rules
local.rules
```

---

## 5. Create the Local Rule

Open the local rules file:

```bash
sudo nano /var/lib/suricata/rules/local.rules
```

Add:

```text
drop icmp any any -> any any (msg:"LAB TEST ICMP BLOCK"; sid:1000001; rev:2;)
```

This rule tells Suricata to drop ICMP traffic.

### Rule Breakdown

```text
drop
```

Tells Suricata to block matching traffic.

```text
icmp
```

Specifies the ICMP protocol.

```text
any any -> any any
```

Matches ICMP traffic from any source to any destination.

```text
msg:"LAB TEST ICMP BLOCK"
```

The message recorded when the rule matches.

```text
sid:1000001
```

Unique Suricata rule ID.

```text
rev:2
```

Rule revision number.

---

# Configure Suricata

## 6. Enable the Local Rule

Open the Suricata configuration:

```bash
sudo nano /etc/suricata/suricata.yaml
```

Find:

```yaml
rule-files:
```

Make sure it contains:

```yaml
rule-files:
  - suricata.rules
  - local.rules
```

Save the file.

---

# Test the Configuration

## 7. Validate Suricata

Before starting the IPS:

```bash
sudo suricata -T -c /etc/suricata/suricata.yaml
```

The configuration should load successfully.

This step verifies that Suricata can read the configuration and rules without errors.

---

# Configure NFQUEUE

## 8. Add the Firewall Rule

On the victim:

```bash
sudo iptables -I INPUT -j NFQUEUE --queue-num 0
```

This sends incoming packets to NFQUEUE number `0`.

Check the rule:

```bash
sudo iptables -L INPUT -n -v --line-numbers
```

You should see an entry similar to:

```text
NFQUEUE num 0
```

---

# Start Suricata IPS

## 9. Start Suricata

Run:

```bash
sudo suricata -c /etc/suricata/suricata.yaml -q 0
```

The important option is:

```text
-q 0
```

This tells Suricata to process packets from NFQUEUE 0.

A successful startup should show:

```text
Engine started.
```

Keep Suricata running while testing.

---

# Test the IPS

## 10. Send ICMP Traffic

On the attacker machine:

```bash
ping 192.168.0.107
```

The packets are sent toward the victim.

The packet flow is:

```text
Attacker
   ↓
Victim
   ↓
iptables INPUT
   ↓
NFQUEUE 0
   ↓
Suricata
   ↓
ICMP rule matches
   ↓
DROP
```

Because the rule uses `drop`, the ICMP packets should be blocked.

---

# Verify Prevention

## 11. Stop Suricata

Return to the victim and press:

```text
Ctrl + C
```

Suricata should display NFQUEUE statistics similar to:

```text
nfq: (RX-NFQ#0) Treated: Pkts 167, Bytes 21752, Errors 0
nfq: (RX-NFQ#0) Verdict: Accepted 27, Dropped 140, Replaced 0
```

The important value is:

```text
Dropped 140
```

This demonstrates that Suricata issued drop verdicts for packets processed through NFQUEUE.

---

## 12. Check Suricata Alerts

Check the alert log:

```bash
sudo cat /var/log/suricata/fast.log
```

Search specifically for the custom rule:

```bash
sudo grep "LAB TEST ICMP BLOCK" /var/log/suricata/fast.log
```

The log should contain an event associated with:

```text
LAB TEST ICMP BLOCK
```

---

## 13. Check EVE JSON

Suricata also records events in:

```text
/var/log/suricata/eve.json
```

Check:

```bash
sudo grep "LAB TEST ICMP BLOCK" /var/log/suricata/eve.json
```

---

# IDS vs IPS

## IDS Mode

When Suricata is started using:

```bash
sudo suricata -i wlan0 -c /etc/suricata/suricata.yaml
```

it operates as a live packet inspection system.

The basic flow is:

```text
Traffic
   ↓
Suricata
   ↓
Detection
   ↓
Alert
```

The traffic itself is not automatically blocked by this setup.

---

## IPS Mode

The IPS configuration uses:

```bash
sudo iptables -I INPUT -j NFQUEUE --queue-num 0
```

and:

```bash
sudo suricata -c /etc/suricata/suricata.yaml -q 0
```

The flow becomes:

```text
Traffic
   ↓
iptables
   ↓
NFQUEUE
   ↓
Suricata
   ↓
Rule Matching
   ↓
┌─────────────┐
│             │
▼             ▼
ACCEPT        DROP
```

This allows Suricata to actively prevent matching traffic from reaching the victim.

---

# Project Structure

The main Suricata configuration and rule files are:

```text
/etc/suricata/
└── suricata.yaml

/var/lib/suricata/rules/
├── classification.config
├── suricata.rules
└── local.rules

/var/log/suricata/
├── eve.json
├── fast.log
├── stats.log
└── suricata.log
```

---

# Useful Commands

### Check Suricata version

```bash
suricata -V
```

### Check configuration

```bash
sudo suricata -T -c /etc/suricata/suricata.yaml
```

### Check network interfaces

```bash
ip -br addr
```

### Check routing

```bash
ip route
```

### Check firewall rules

```bash
sudo iptables -L INPUT -n -v --line-numbers
```

### Check Suricata alerts

```bash
sudo cat /var/log/suricata/fast.log
```

### Monitor alerts live

```bash
sudo tail -f /var/log/suricata/fast.log
```

### Monitor EVE JSON

```bash
sudo tail -f /var/log/suricata/eve.json
```

---

# Disable the IPS

Stop Suricata:

```text
Ctrl + C
```

Remove the NFQUEUE rule:

```bash
sudo iptables -D INPUT -j NFQUEUE --queue-num 0
```

Verify:

```bash
sudo iptables -L INPUT -n -v --line-numbers
```

---

# Current Implementation

The current version successfully implements:

* [x] Suricata installation
* [x] Suricata configuration
* [x] Custom Suricata rule
* [x] ICMP traffic detection
* [x] Linux iptables integration
* [x] NFQUEUE configuration
* [x] Suricata NFQUEUE mode
* [x] Packet inspection
* [x] Packet dropping
* [x] IPS logging
* [x] Attacker-to-victim IPS testing

---

# Future Improvements

Possible future improvements include:

* Additional detection rules
* Detection of TCP-based attacks
* Detection of UDP-based attacks
* Detection of port scanning
* Custom attack signatures
* Automated IP blocking
* More extensive attack simulations
* Performance testing
* False-positive analysis
* IPS logging dashboard
* Rule management automation

---

# Disclaimer

This project is intended for **educational, research, and authorized security-testing purposes only**.

Testing should be performed only on systems and networks where you have permission to conduct security testing.

---

# Author

**Tamim Haider**

Computer Science & Engineering
Cybersecurity / Network Security Project
