# 📖 SOC Lab Simulator & Threat Detection Documentation

## 1. Executive Summary & Architecture Overview

This project documents the design, implementation, and adversary testing of a localized **Security Operations Center (SOC)** detection platform. The infrastructure monitors hybrid operating system endpoints (Windows 10 and Ubuntu Linux), ingests kernel-level process and authentication events into a centralized **Wazuh SIEM Manager** hosted on Docker, tests detection logic using **Atomic Red Team**, and executes automated host containment via **Active Response**.

### Topology Model
* **Manager Node:** Wazuh SIEM 4.8.x Single-Node Cluster (Docker Desktop on Windows Host).
* **Endpoint 1 (Local Windows):** Wazuh Agent + Microsoft Sysmon (Modular Configuration).
* **Endpoint 2 (Remote Ubuntu Laptop):** Wazuh Agent + Linux Auditd Subsystem.
* **Adversary Simulation:** Red Canary Atomic Red Team (MITRE ATT&CK TTP execution).

       +-------------------------------------------------------+
       |              Wazuh SIEM Manager (Docker)              |
       |           [ Threat Hunting & Mitre Matrix ]           |
       +---------------------------^---------------------------+
                                   | (Port 1514/1515 TCP TLS)
                 +-----------------+-----------------+
                 |                                   |
       +---------+-----------+             +---------+-----------+
       |   Windows 10 Host   |             |   Ubuntu Workstation|
       | Wazuh Agent + Sysmon|             | Wazuh Agent + Auditd|
       +---------------------+             +---------------------+

---

## 2. SIEM Manager Deployment (Docker)

1. Clone the production single-node deployment repository:

        git clone https://github.com/wazuh/wazuh-docker.git -b v4.8.0 --depth=1
        cd wazuh-docker/single-node

2. Generate internal self-signed indexing certificates:

        docker compose -f generate-indexing-certs.yml run --rm generator

3. Initialize the Wazuh cluster containers:

        docker compose up -d

4. Configure the Windows Host Firewall to permit agent communications:

        New-NetFirewallRule -DisplayName "Wazuh Agent Comm" -Direction Inbound -Protocol TCP -LocalPort 1514,1515 -Action Allow

---

## 3. Endpoint Ingestion & Agent Enrollment

### A. Windows 10 Host Onboarding
1. Install **Microsoft Sysmon** with a modular configuration to capture Process Creation (`Event ID 1`) and Registry Modification (`Event ID 13`):

        sysmon64.exe -accepteula -i sysmonconfig.xml

2. Download and deploy the Windows Wazuh Agent pointing to the host IP:

        Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.8.0-1.msi -OutFile ${env:tmp}\wazuh-agent.msi
        msiexec.exe /i ${env:tmp}\wazuh-agent.msi /q WAZUH_MANAGER='<MANAGER_IP>' WAZUH_AGENT_NAME='Win10-Victim-Lab'
        NET START WazuhSvc

### B. Remote Ubuntu Laptop Onboarding
1. Enable the native Linux auditing daemon for command-level telemetries:

        sudo apt update && sudo apt install -y auditd
        sudo systemctl enable --now auditd

2. Register and start the Ubuntu Wazuh Agent:

        wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.8.0-1_amd64.deb
        sudo WAZUH_MANAGER='<MANAGER_IP>' WAZUH_AGENT_NAME='Ubuntu-WorkStation-01' dpkg -i ./wazuh-agent_4.8.0-1_amd64.deb
        sudo systemctl daemon-reload
        sudo systemctl enable --now wazuh-agent

---

## 4. Adversary Emulation & MITRE ATT&CK Testing

### Technique 1: System Owner / Discovery (`T1033` / `T1087.001`)
* **Vector:** Enumeration of user context and sensitive account files.
* **Execution (Ubuntu):**

        cat /etc/passwd
        cat /etc/shadow
        whoami

* **SIEM Detection:** Triggered `pam` session logs and File Integrity Monitoring events for sensitive path reads.

### Technique 2: Persistence via Scheduled Cron (`T1053.003`)
* **Vector:** Registering arbitrary recurring execution via crontab.
* **Execution (Ubuntu):**

        (crontab -l 2>/dev/null; echo "* * * * * /tmp/evil.sh") | crontab -
        crontab -r

* **SIEM Detection:** Ingested under `crontab` modification events, classified under MITRE Persistence.

### Technique 3: Privilege Escalation via Sudoers Tampering (`T1548.003`)
* **Vector:** Granting root privileges by appending arbitrary rules into `/etc/sudoers.d/`.
* **Execution (Ubuntu):**

        echo "attacker ALL=(ALL) NOPASSWD: ALL" | sudo tee -a /etc/sudoers.d/attacker
        sudo rm -f /etc/sudoers.d/attacker

* **SIEM Detection:** Wazuh flagged authentication privilege escalation and PAM session escalation (Rule ID `5402` & `5403`).

---

## 5. Automated Incident Response (Active Response)

To enforce containment against persistent privilege abuse or brute-force activities, an Active Response loop was configured inside `/var/ossec/etc/ossec.conf` on the Wazuh Manager:

    <active-response>
      <command>firewall-drop</command>
      <location>local</location>
      <rules_id>5402, 5710, 5712</rules_id>
      <timeout>60</timeout>
    </active-response>

### Verification & Telemetry Output
* **Trigger:** Multiple failed `sudo` commands executed in rapid succession on the Ubuntu endpoint.
* **Containment Execution:** The agent executed the `firewall-drop` script, appending an automated `iptables` drop rule for 60 seconds.
* **Log Verification (Ubuntu):**

        sudo tail -n 20 /var/ossec/logs/active-responses.log

---

## 6. Telemetry & Visual Evidence

* **SOC Overview Metrics:**
  ![Wazuh Overview](screenshots/overview_2.png)

* **MITRE ATT&CK Matrix Mapping (Ubuntu Workstation):**
  ![MITRE Dashboard](screenshots/MITTRE_2.png)

* **Threat Hunting & Event Correlation:**
  ![Threat Hunting Logs](screenshots/threat-hunting_2.png)