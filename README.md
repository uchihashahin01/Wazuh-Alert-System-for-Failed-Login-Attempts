### Step 1: Install Wazuh Manager, Indexer, and Dashboard

1. **Download and Run the Installation Script**
    - Run:
        
        ```bash
        curl -sO <https://packages.wazuh.com/4.11/wazuh-install.sh> && sudo bash ./wazuh-install.sh -a
        ```
        
        - This installs the Wazuh manager, indexer, and dashboard.
        - Remember to copy the admin name and password. (it will be needed later)
2. **Verify Services**
    - Check:
        
        ```bash
        sudo systemctl status wazuh-manager
        sudo systemctl status wazuh-indexer
        sudo systemctl status wazuh-dashboard
        ```
        
        - Ensure all are “active (running)”.
3. **Get VM’s IP Address**
    - Run:
        
        ```bash
        hostname -I
        ```
        
        - Note the IP (e.g., `192.168.8.157`).

---

### Step 2: Configure SSH for Failed Login Testing

1. **Install OpenSSH Server**
    - Run:
        
        ```bash
        sudo apt update
        sudo apt install openssh-server -y
        ```
        
2. **Enable and Start SSH**
    - Run:
        
        ```bash
        sudo systemctl enable ssh
        sudo systemctl start ssh
        ```
        
    - Verify:
        
        ```bash
        sudo systemctl status ssh
        ```
        
3. **Configure SSH Logging**
    - Edit:
        
        ```bash
        sudo nano /etc/ssh/sshd_config
        ```
        
    - Uncomment or set:
        
        ```
        SyslogFacility AUTH
        LogLevel INFO
        ```
        
    - Save (`Ctrl+X`, `y`, `enter`) and restart:
        
        ```bash
        sudo systemctl restart ssh
        ```
        
4. **Test Failed Logins**
    - Run:
        
        ```bash
        ssh wronguser@localhost
        ```
        
        - Enter a random password, repeat 2–3 times.
    - Check:
        
        ```bash
        sudo tail -n 10 /var/log/auth.log
        ```
        
        - Expect:
            
            ```
            Failed password for invalid user wronguser from 127.0.0.1 port 12345 ssh2
            ```
            

---

### Step 3: Configure Wazuh Manager to Monitor auth.log

The Wazuh manager will directly monitor `/var/log/auth.log`.

1. **Edit ossec.conf**
    - Open:
        
        ```bash
        sudo nano /var/ossec/etc/ossec.conf
        ```
        
    - Add this line before the ending tag of`</ossec_config>`:
        
        ```xml
        <localfile>
          <log_format>syslog</log_format>
          <location>/var/log/auth.log</location>
        </localfile>
        ```
        
        - Save (`Ctrl+X`, `y`, `enter`).
2. **Set Permissions**
    - Run:
        
        ```bash
        sudo chmod 644 /var/log/auth.log
        sudo chown :wazuh /var/log/auth.log
        ```
        
3. **Restart Wazuh Manager**
    - Run:
        
        ```bash
        sudo systemctl restart wazuh-manager
        ```
        
4. **Verify Log Collection**
    - Generate failed logins:
        
        ```bash
        for i in {1..5}; do ssh wronguser@localhost false; done
        ```
        
    - Check:
        
        ```bash
        sudo tail -n 20 /var/ossec/logs/alerts/alerts.log
        ```
        
        - Look for `/var/log/auth.log` activity in wazuh logs

---

### Step 4: Create a Custom Wazuh Rule

1. **Edit the Rules File**
    - Open:
        
        ```bash
        sudo nano /var/ossec/etc/rules/local_rules.xml
        ```
        
    - Replace with:
        
        ```xml
        <!-- Local rules -->
        <!-- Copyright (C) 2015, Wazuh Inc. -->
        <group name="local,syslog,authentication,">
          <rule id="100001" level="7">
            <if_sid>5710</if_sid>
            <description>Multiple failed login attempts detected</description>
            <mitre>
              <id>T1110</id>
            </mitre>
            <group>authentication_failures,</group>
          </rule>
        </group>
        ```
        
        - Save (`Ctrl+X`, `y`, `enter`).
2. **Set Permissions**
    - Run:
        
        ```bash
        sudo chown wazuh:wazuh /var/ossec/etc/rules/local_rules.xml
        sudo chmod 640 /var/ossec/etc/rules/local_rules.xml
        ```
        
3. **Restart Wazuh Manager**
    - Run:
        
        ```bash
        sudo systemctl restart wazuh-manager
        ```
        
4. **Test the Rule and Check Alerts Log**
    - Generate failed logins:
        
        ```bash
        for i in {1..5}; do ssh wronguser@localhost false; done
        ```
        
    - Check Wazuh alerts log:
        
        ```bash
        sudo tail -n 20 /var/ossec/logs/alerts/alerts.log
        ```
        
        - Expect entries like:
            
            ```
            ** Alert 1745524475.193183: - local,syslog,authentication,authentication_failures,
            2025 Apr 24 15:54:35 osboxes->journald
            Rule: 100001 (level 7) -> 'Multiple failed login attempts detected'
            Src IP: 127.0.0.1
            
            ```
            
        - Also look for `rule id="5710"` (default SSH failure rule).

---

### Step 5: Create a Visualizer in the Wazuh Dashboard

1. **Access the Dashboard**
    - Open:
        
        ```
        https://<your-vm-ip>
        ```
        
        - Example: `https://192.168.8.157`.
        - Log in with `admin`/<password>.
2. **Go to Visualize Section**
    - Click the hamburger menu (top-left).
    - Go to **Visualize** > **Create visualization**.
    - Select **Data Table**.
3. **Choose Data Source**
    - In “New Data Table / Choose a source”, select `wazuh-alerts-*`.
    - **Error Fix**: If missing:
        
        ```bash
        sudo systemctl restart wazuh-indexer
        ```
        
4. **Configure the Data Table**
    - **Add Filter**:
        - Click “Add filter”.
        - Add: `rule.id:100001`.
        - Optionally add: `authentication_failed`, `wronguser`, `src.ip:127.0.0.1`.
    - **Set Time Range**:
        - Set to “Last 24 hours” or “Last hour” (top-right) to include your alerts.
    - **Add Columns (Buckets)**:
        - Under “Buckets” (right panel), click “Add” > “Split rows” > “Terms”.
        - Add fields:
            - `timestamp`
            - `rule.id`
            - `rule.description`
            - `src.ip`
            - `src.port`
            - `user.name`
            - `full_log`
        - Click “Update” after each addition.
    - **Verify Table**:
        - The table should show alerts with columns (e.g., `wronguser`, `127.0.0.1`, `rule.id:100001`).
5. **Save the Visualization**
    - Click “Save” (top-right).
    - Name it (e.g., “Failed Login Table”).
    - Save.
6. **Add to Dashboard**
    - Go to **Dashboards** > **Create Dashboard**.
    - Add the “Failed Login Table” visualization.
    - Save the dashboard (e.g., “Security Alerts Dashboard”).

---