# AWS Cisco Logging Dashboard — Step 1 (Fresh EC2 instance + networking + security)

Goal: launch a clean EC2 instance that can receive **Cisco syslog** (UDP/514) and **SNMP traps** (UDP/162) and host the **Grafana UI** (TCP/3000).

This design works with Verizon Home Internet / CGNAT because your Cisco devices **push traffic outbound** to AWS.

---

## 1) Choose region
Pick a single AWS region and use it consistently (example: **us-east-1**).

---

## 2) Create a Security Group (SG) first
EC2 → **Security Groups** → **Create security group**

Name example: `sg-cisco-logging-lab`

### Inbound rules (recommended)
Use your **home public IPv4** as the source (best practice).

To find it quickly from your PC:
- Search “what is my IP” in a browser, or use:
  ```bash
  curl -4 ifconfig.me
  ```

Add inbound rules:

1) **SSH (admin access)**
- Type: SSH
- Protocol: TCP
- Port: 22
- Source: `<YOUR_HOME_PUBLIC_IP>/32`

2) **Syslog from Cisco**
- Type: Custom UDP
- Port: 514
- Source: `<YOUR_HOME_PUBLIC_IP>/32`

3) **SNMP traps from Cisco**
- Type: Custom UDP
- Port: 162
- Source: `<YOUR_HOME_PUBLIC_IP>/32`

4) **Grafana Web UI**
- Type: Custom TCP
- Port: 3000
- Source: `<YOUR_HOME_PUBLIC_IP>/32`

### Temporary “it’s not working yet” option
If you cannot determine your home public IP or it changes frequently, you can temporarily set the syslog/trap rules to `0.0.0.0/0` for testing, then tighten them back to your IP once confirmed.

---

## 3) Launch a fresh EC2 instance
EC2 → **Instances** → **Launch instances**

Recommended settings for a lab:

### Name
- `cisco-logging-lab`

### AMI
- **Amazon Linux 2023**

### Instance type
- `t3.micro` (or another free-tier-eligible micro instance for your account)

### Key pair
- Create or select a key pair you control.
- Download the `.pem` file and store it safely (you’ll SSH with it).

### Network settings
- VPC: default is fine for a lab
- Subnet: any default subnet
- **Auto-assign public IP: Enabled**
- Firewall (Security Group): select the SG you created above (`sg-cisco-logging-lab`)

### Storage
- Default is fine (8–30 GB). Logs can grow; if you plan to keep logs for days/weeks, consider 20+ GB.

Launch the instance.

---

## 4) Record the connection details you will reuse
From the instance details page, copy:

- **Public IPv4 address** (example: `3.x.x.x`)
- (Optional) **Public IPv4 DNS** name

You will use the public IPv4 address for:
- Cisco `logging host <EC2_PUBLIC_IP>`
- Cisco `snmp-server host <EC2_PUBLIC_IP> ...`
- Grafana URL: `http://<EC2_PUBLIC_IP>:3000`

Note: If you stop/start the instance, the public IP can change. For a short lab, that’s usually fine—just update the Cisco configs. If you want a stable public IP, you can use an Elastic IP, but that may incur cost depending on how it’s used.

---

## 5) SSH into the instance (quick test)
From your PC:

```bash
ssh -i /path/to/key.pem ec2-user@<EC2_PUBLIC_IP>
```

If SSH fails:
- Confirm your SG inbound rule allows TCP/22 from your home IP
- Confirm you used the correct username (`ec2-user` for Amazon Linux)
- Confirm the instance is in “running” state and has a public IP

---

## 6) Cleanup (after the lab)
To avoid charges:
- Terminate the instance when finished (EC2 → Instances → Actions → Instance state → Terminate)
- If you allocated an Elastic IP, release it
- Delete the security group if you no longer need it

---

## Next
Proceed to **Steps 2 & 3** to configure:
- rsyslog (UDP/514 receiver)
- snmptrapd (UDP/162 receiver)
