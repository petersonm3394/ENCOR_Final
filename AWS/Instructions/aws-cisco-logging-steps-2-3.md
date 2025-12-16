# AWS Cisco Logging Dashboard — Steps 2 & 3 (EC2 host services: syslog + SNMP traps)

These instructions assume you already completed **Step 1** (fresh EC2 instance + Security Group).

## Step 1 prerequisites (quick check)
Your EC2 Security Group should allow inbound from your public IP (recommended):
- **TCP 22** (SSH)
- **UDP 514** (syslog from Cisco)
- **UDP 162** (SNMP traps from Cisco)

Grafana (TCP 3000) is handled later in the Grafana setup file.

---

## Step 2 — SSH in and install prerequisites

### 2.1 SSH to the instance
From your PC:

```bash
ssh -i /path/to/key.pem ec2-user@<EC2_PUBLIC_IP>
```

### 2.2 Update OS and install packages
On the EC2 instance:

```bash
sudo dnf update -y
sudo dnf install -y rsyslog curl
```

---

## Step 3 — Configure rsyslog to receive Cisco syslog into a file

### 3.1 Create an rsyslog rule for UDP syslog (port 514)
Create a dedicated config file:

```bash
sudo nano /etc/rsyslog.d/10-cisco-syslog.conf
```

Paste:

```conf
# Listen for syslog over UDP/514
module(load="imudp")
input(type="imudp" port="514")

# For this lab: put all UDP syslog into one file
if $inputname == "imudp" then /var/log/cisco-lab.log
& stop
```

### 3.2 Enable + restart rsyslog
```bash
sudo systemctl enable rsyslog
sudo systemctl restart rsyslog
```

### 3.3 Confirm rsyslog is listening on UDP/514
```bash
sudo ss -lunp | grep :514
```

### 3.4 Verify the log file exists (it will populate once traffic arrives)
```bash
sudo ls -l /var/log/cisco-lab.log || true
```

---

## Step 3B — (Optional but recommended) Receive SNMP traps and store in a file

This enables “event-based monitoring” (linkUp/linkDown, auth failures, etc.) without AWS needing inbound access to your home network beyond UDP/162.

### 3B.1 Install SNMP trap receiver
```bash
sudo dnf install -y net-snmp net-snmp-utils
```

### 3B.2 Configure snmptrapd community authorization
Edit:

```bash
sudo nano /etc/snmp/snmptrapd.conf
```

Add (choose your own community string):

```conf
authCommunity log,execute,net TRAPCOMM123
```

Restart and enable snmptrapd:

```bash
sudo systemctl enable --now snmptrapd
sudo systemctl restart snmptrapd
```

Confirm it is listening on UDP/162:

```bash
sudo ss -lunp | grep :162
```

### 3B.3 Route snmptrapd logs into a dedicated file
Create an rsyslog rule:

```bash
sudo nano /etc/rsyslog.d/20-snmptraps.conf
```

Paste:

```conf
if $programname == 'snmptrapd' then /var/log/snmptraps.log
& stop
```

Restart rsyslog:

```bash
sudo systemctl restart rsyslog
```

Verify trap log file exists (will populate once traps arrive):

```bash
sudo ls -l /var/log/snmptraps.log || true
```

---

## Cisco device configuration (required to generate syslog + traps)

### A) Make each device identify itself uniquely in logs (multiple devices)
On **each** Cisco device:

```text
conf t
 hostname R1                  ! Use unique names: R1, R2, SW1, etc.
 logging origin-id hostname
 service timestamps log datetime msec localtime
 service sequence-numbers
end
```

### B) Send syslog to EC2 (UDP 514)
On each Cisco device:

```text
conf t
 logging trap informational
 logging host <EC2_PUBLIC_IP> transport udp port 514
 ! Optional: choose your LAN interface as the source address for syslog packets
 logging source-interface <LAN_INTERFACE>
end
```

### C) Send SNMP traps to EC2 (UDP 162)
On each Cisco device (SNMPv2c trap example):

```text
conf t
 snmp-server community TRAPCOMM123 RO
 snmp-server enable traps
 snmp-server enable traps snmp linkdown linkup
 snmp-server host <EC2_PUBLIC_IP> version 2c TRAPCOMM123
end
```

---

## Quick “does it work?” checks

### Generate syslog from Cisco
Flap an interface (safe lab interface):

```text
conf t
 interface <TEST_INTERFACE>
  shutdown
  no shutdown
end
```

### Confirm syslog arrived on EC2
```bash
sudo tail -n 50 /var/log/cisco-lab.log
```

### Confirm traps arrived on EC2
```bash
sudo tail -n 50 /var/log/snmptraps.log
```

If nothing arrives:
- Re-check EC2 Security Group inbound UDP 514 and UDP 162
- Confirm the Cisco device can reach `<EC2_PUBLIC_IP>`
- Confirm `ss -lunp | grep :514` and `ss -lunp | grep :162` show listeners
