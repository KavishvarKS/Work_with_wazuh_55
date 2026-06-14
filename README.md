# 🛡️ Working with Wazuh

> **Wazuh** is a free, open-source security platform that unifies XDR and SIEM capabilities — providing threat detection, integrity monitoring, incident response, and compliance in one place.

📖 **Official Quickstart Docs:** [documentation.wazuh.com/current/quickstart.html](https://documentation.wazuh.com/current/quickstart.html)

---

## 📐 Architecture Overview

```
┌─────────────────────────────┐        ┌──────────────────────┐
│        SERVER NODE          │        │      TEST NODE       │
│                             │        │                      │
│  ┌─────────────────────┐    │◄──────►│  ┌────────────────┐  │
│  │   Wazuh Manager     │    │        │  │  Wazuh Agent   │  │
│  │   Wazuh Indexer     │    │        │  └────────────────┘  │
│  │   Wazuh Dashboard   │    │        │                      │
│  └─────────────────────┘    │        │  Sends logs/events   │
│                             │        │  to server           │
│  Monitors everything        │        └──────────────────────┘
└─────────────────────────────┘

Ports used:  1514/tcp (agent comms)  |  1515/tcp (agent enrollment)  |  443/tcp (dashboard)
```

---

## ⚙️ Requirements

| Component | Minimum |
|-----------|---------|
| OS | Ubuntu 20.04 / 22.04 / 24.04 (64-bit) |
| RAM | 4 GB (server), 512 MB (agent) |
| CPU | 2 cores (server) |
| Disk | 50 GB (server) |
| Network | Public/private IP reachable between nodes |

---

## 🖥️ PART 1 — Server Node Installation

### Step 1 — Update the system

```bash
sudo apt update && sudo apt upgrade -y
```

### Step 2 — Download the Wazuh installation assistant

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
```

### Step 3 — Run the all-in-one installer

```bash
sudo bash ./wazuh-install.sh -a
```

> **What `-a` does:** Installs everything on one machine — Wazuh Manager, Wazuh Indexer, and Wazuh Dashboard — in a single command.

⏳ This will take a few minutes. When it finishes you'll see:

```
INFO: --- Summary ---
INFO: You can access the web interface https://<WAZUH_DASHBOARD_IP>
    User: admin
    Password: <ADMIN_PASSWORD>
INFO: Installation finished.
```

> ⚠️ **Save your password immediately!** You'll need it to log into the dashboard.

---

### Step 4 — Open required firewall ports

```bash
sudo ufw allow 1514/tcp    # Agent log forwarding
sudo ufw allow 1515/tcp    # Agent enrollment/registration
sudo ufw allow 55000/tcp   # Wazuh API
sudo ufw allow 443/tcp     # Dashboard (HTTPS)
sudo ufw reload
```

> ☁️ **If you're on a cloud VM (AWS/GCP/Azure/etc.):** Also open these ports in your cloud provider's **Security Group / Firewall Rules**.

---

### Step 5 — Verify all services are running

```bash
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-dashboard
```

All three should show `active (running)` ✅

---

### Step 6 — Access the Dashboard

Open your browser and go to:

```
https://<YOUR_SERVER_PUBLIC_IP>
```

- **Username:** `admin`
- **Password:** *(the one shown after installation)*

> 🔒 Your browser may warn about an untrusted certificate — this is expected. The installer uses a self-signed certificate. Click **Advanced → Accept / Proceed** to continue.

---

### 💡 Retrieve passwords anytime

If you lose the password, extract it from the install archive:

```bash
sudo tar -O -xvf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt
```

---

## 🤖 PART 2 — Test Node (Agent) Installation

Run these commands on your **second/test node**.

### Step 1 — Add the Wazuh repository

```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo apt-key add -

echo "deb https://packages.wazuh.com/4.x/apt/ stable main" | \
  sudo tee /etc/apt/sources.list.d/wazuh.list

sudo apt update
```

### Step 2 — Install the Wazuh agent

```bash
WAZUH_MANAGER="<SERVER_IP>" sudo apt install wazuh-agent -y
```

> Replace `<SERVER_IP>` with your server node's IP address (public or private depending on your network).

### Step 3 — Configure the agent

Open the agent config file:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Find this block and update the `<address>` field:

```xml
<client>
  <server>
    <address>YOUR_SERVER_IP</address>   <!-- Replace this -->
    <port>1514</port>
    <protocol>tcp</protocol>
  </server>
</client>
```

### Step 4 — Enable and start the agent

```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

### Step 5 — Register the agent with the server

```bash
sudo /var/ossec/bin/agent-auth -m <SERVER_IP>
```

### Step 6 — Check agent status

```bash
sudo systemctl status wazuh-agent
```

---

## ✅ PART 3 — Verify Connection

### On the Server — list connected agents

```bash
sudo /var/ossec/bin/agent_control -l
```

Expected output:

```
ID: 001, Name: test-node, IP: <TEST_NODE_IP>, Active
```

### On the Dashboard

1. Go to `https://<SERVER_IP>`
2. Navigate to **Wazuh → Agents**
3. Your test node should appear with status **Active** ✅

---

## 🧪 PART 4 — Test It

Generate some log activity on the test node:

```bash
# View auth logs
cat /var/log/auth.log

# Simulate a failed SSH login (triggers a security alert)
ssh wronguser@localhost
```

Then in the dashboard go to **Security Events** — you should see alerts coming in from your agent in real time.

---

## 🔍 Useful Diagnostic Commands

```bash
# Check open ports on server
sudo ss -tulnp | grep -E '1514|1515|443|9200'

# View live server logs
sudo tail -f /var/ossec/logs/ossec.log

# View dashboard logs
sudo journalctl -u wazuh-dashboard -n 50 --no-pager

# View nginx error logs (if using reverse proxy)
sudo tail -n 20 /var/log/nginx/error.log

# Check agent status from server
sudo /var/ossec/bin/agent_control -i <AGENT_ID>
```

---

## ❌ Common Issues & Fixes

| Problem | Fix |
|--------|-----|
| Agent not showing in dashboard | Check firewall — ports 1514/1515 must be open |
| Connection refused | Verify correct server IP in `ossec.conf` |
| Agent shows Inactive | Run `sudo systemctl restart wazuh-agent` |
| Dashboard not loading | Run `sudo systemctl restart wazuh-dashboard` |
| Certificate warning in browser | Expected — accept the self-signed cert exception |

---

## 🗑️ Uninstall Wazuh

To completely remove all Wazuh central components:

```bash
sudo bash wazuh-install.sh -u
```

Or use the `--uninstall` flag:

```bash
sudo bash wazuh-install.sh --uninstall
```

---

## 📚 References

- 📖 [Official Quickstart Guide](https://documentation.wazuh.com/current/quickstart.html)
- 📖 [Agent Enrollment Docs](https://documentation.wazuh.com/current/user-manual/agent/agent-enrollment/index.html)
- 📖 [Wazuh Blog — Email Notifications](https://wazuh.com/blog/how-to-send-email-notifications-with-wazuh/)

---

*Made with 💙 — Wazuh open-source security monitoring*
