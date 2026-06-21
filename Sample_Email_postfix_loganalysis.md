# 📧 Mail Server Setup for Wazuh Log Analysis Testing

A step-by-step guide to setting up two Postfix mail servers on Ubuntu 24.04 and sending emails between them — designed for Wazuh SIEM log analysis testing and learning.

---

## 🖥️ Lab Architecture

```
Node A (Sender)                        Node B (Receiver)
┌──────────────────────┐              ┌──────────────────────┐
│  Ubuntu 24.04 LTS    │              │  Ubuntu 24.04 LTS    │
│  Postfix (SMTP)      │──── SMTP ───►│  Postfix (SMTP)      │
│  mailutils           │   Port 25    │  Maildir storage     │
│  hostname: nodea     │              │  hostname: nodeb     │
└──────────────────────┘              └──────────────────────┘
   Public IP: <Node-A-IP>                Public IP: <Node-B-IP>
```

> This setup is intentionally simple — no TLS, no auth — purely for internal lab/Wazuh testing.

---

## ✅ Prerequisites

- 2x Ubuntu 24.04 LTS servers (cloud VMs, local VMs, or bare metal)
- Root or sudo access on both
- Port 25 open between both nodes
- Both nodes reachable via public or private IP

---

## 📦 Step 1 — Update Both Nodes

Run on **both nodes**:

```bash
sudo apt update && sudo apt upgrade -y
```

---

## 📬 Step 2 — Install Postfix on Both Nodes

```bash
sudo apt install postfix mailutils -y
```

During installation:
- When prompted → select **"Internet Site"**
- System mail name:
  - Node A → press Enter (accept default)
  - Node B → press Enter (accept default)

---

## ⚙️ Step 3 — Configure Node A (Sender)

```bash
postconf -e "myhostname = nodea.local"
postconf -e "mydestination = \$myhostname, nodea.local, localhost.localdomain, localhost"
postconf -e "mynetworks = <YOUR-NETWORK-CIDR> 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128"
postconf -e "home_mailbox = Maildir/"
postconf -e "inet_protocols = ipv4"
```

Add transport map to bypass DNS and route directly to Node B:

```bash
cat >> /etc/postfix/transport << 'EOF'
nodeb.local smtp:[<NODE-B-PUBLIC-IP>]:25
EOF

postmap /etc/postfix/transport
postconf -e "transport_maps = hash:/etc/postfix/transport"
systemctl restart postfix
```

---

## ⚙️ Step 4 — Configure Node B (Receiver)

```bash
postconf -e "myhostname = nodeb.local"
postconf -e "mydestination = \$myhostname, nodeb.local, localhost.localdomain, localhost, <NODE-B-PUBLIC-IP>"
postconf -e "mynetworks = <YOUR-NETWORK-CIDR> 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128"
postconf -e "home_mailbox = Maildir/"
postconf -e "inet_protocols = ipv4"
systemctl restart postfix
```

---

## 👤 Step 5 — Create Mail User on Node B

```bash
useradd -m -s /bin/bash mailuser
echo "mailuser:test123" | chpasswd

# Create Maildir structure
mkdir -p /home/mailuser/Maildir/{new,cur,tmp}
chown -R mailuser:mailuser /home/mailuser/Maildir
chmod -R 700 /home/mailuser/Maildir
```

---

## 📝 Step 6 — Enable Mail Logging on Both Nodes

Ubuntu 24.04 does not create `/var/log/mail.log` automatically. Fix this on **both nodes**:

```bash
apt install rsyslog -y
systemctl enable rsyslog
systemctl start rsyslog

touch /var/log/mail.log
chown syslog:adm /var/log/mail.log
chmod 640 /var/log/mail.log

systemctl restart rsyslog
systemctl restart postfix
```

---

## 🔗 Step 7 — Add Node B to Node A's Hosts File

On **Node A only**:

```bash
echo "<NODE-B-PUBLIC-IP> nodeb.local" >> /etc/hosts

# Also copy into Postfix chroot jail
cp /etc/hosts /var/spool/postfix/etc/hosts
cp /etc/nsswitch.conf /var/spool/postfix/etc/nsswitch.conf

systemctl restart postfix
```

> ⚠️ **Important:** Postfix on Ubuntu runs inside a chroot jail at `/var/spool/postfix/`. It does NOT read the system `/etc/hosts` — you must copy it into the chroot.

---

## 📤 Step 8 — Send a Test Email

On **Node A**:

```bash
echo "Hello from Node A!" | mail -s "Test Email" mailuser@nodeb.local
```

---

## 📥 Step 9 — Verify Email Received on Node B

```bash
ls -la /home/mailuser/Maildir/new/
```

You should see a file like:
```
-rw------- 1 mailuser mailuser 630 Jun 21 15:35 1782036345.Vfd02I12400088M742982.e2e-79-194
```

Read the email:
```bash
cat /home/mailuser/Maildir/new/<filename>
```

---

## 🔍 Step 10 — Log Analysis

### Node A — Sender Logs

```bash
tail -f /var/log/mail.log
```

### Node B — Receiver Logs

```bash
tail -f /var/log/mail.log
```

### Trace a specific email by Queue ID

```bash
grep "<QUEUE-ID>" /var/log/mail.log
```

---

## 📖 Understanding Mail Log Fields

```
2026-06-21T15:35:45+05:30 nodea postfix/smtp[23422]: 9FBB4A0000C5:
  to=<mailuser@nodeb.local>,
  relay=164.52.210.194[164.52.210.194]:25,
  delay=0.09,
  delays=0.02/0.02/0.04/0.01,
  dsn=2.0.0,
  status=sent (250 2.0.0 Ok: queued as B39B890000C6)
```

| Field | Meaning |
|---|---|
| `postfix/smtp[23422]` | SMTP process and PID |
| `9FBB4A0000C5` | Unique mail Queue ID |
| `to=` | Recipient address |
| `relay=` | Server that accepted the mail |
| `delay=0.09` | Total delivery time in seconds |
| `delays=0.02/0.02/0.04/0.01` | Time in: queue / connect / transfer / done |
| `dsn=2.0.0` | Delivery Status Notification — 2.x.x = success |
| `status=sent` | Final delivery status |
| `250 2.0.0 Ok` | SMTP response from receiving server |
| `queued as B39B8...` | Node B's internal queue ID for this mail |

### Email Lifecycle in Logs

```
pickup   → Mail picked up from submission queue
cleanup  → Headers added, message ID assigned
qmgr     → Queued for delivery
smtp     → Delivered to remote server
delivered → Placed in recipient's Maildir
```

### Common Status Codes

| DSN Code | Meaning |
|---|---|
| `2.0.0` | ✅ Delivered successfully |
| `4.x.x` | ⏳ Temporary failure (will retry) |
| `5.1.3` | ❌ Bad address syntax |
| `5.4.4` | ❌ Host not found (DNS/routing issue) |
| `5.7.1` | ❌ Relay denied |

---

## 🐛 Troubleshooting — Issues We Hit & Fixes

### Issue 1 — `type=AAAA: Host not found`
**Cause:** Postfix defaulting to IPv6 DNS lookup.  
**Fix:**
```bash
postconf -e "inet_protocols = ipv4"
systemctl restart postfix
```

### Issue 2 — `bad address syntax`
**Cause:** Sending to a raw IP address like `mailuser@192.168.1.1` — Postfix doesn't accept this.  
**Fix:** Always use a hostname: `mailuser@nodeb.local`

### Issue 3 — `type=A: Host not found` (even with /etc/hosts set)
**Cause:** Postfix runs in a **chroot jail** and ignores the system `/etc/hosts`.  
**Fix:**
```bash
cp /etc/hosts /var/spool/postfix/etc/hosts
systemctl restart postfix
```

### Issue 4 — `/var/log/mail.log` does not exist
**Cause:** Ubuntu 24.04 uses journald by default; rsyslog doesn't create the file automatically.  
**Fix:**
```bash
touch /var/log/mail.log
chown syslog:adm /var/log/mail.log
chmod 640 /var/log/mail.log
systemctl restart rsyslog
```

### Issue 5 — Maildir doesn't exist
**Cause:** Postfix won't auto-create Maildir for new users.  
**Fix:**
```bash
mkdir -p /home/mailuser/Maildir/{new,cur,tmp}
chown -R mailuser:mailuser /home/mailuser/Maildir
```

### Issue 6 — Mail still not routing after all fixes
**Fix:** Use a transport map to bypass DNS entirely:
```bash
echo "nodeb.local smtp:[<NODE-B-IP>]:25" >> /etc/postfix/transport
postmap /etc/postfix/transport
postconf -e "transport_maps = hash:/etc/postfix/transport"
systemctl restart postfix
```

---

## 🔐 Wazuh Integration Notes

Once mail is flowing between nodes, Wazuh can monitor:

- `/var/log/mail.log` on both nodes for mail events
- Bounce patterns → detect delivery failures
- Unusual sender/recipient patterns → detect spam or exfiltration
- High mail volume spikes → potential abuse

### Suggested Wazuh Rules to Test

| Scenario | What to trigger |
|---|---|
| Brute force via mail | Send many emails rapidly from Node A |
| Bounce storm | Send to non-existent users |
| Relay attempt | Try sending from an unauthorized IP |
| Large attachment | Send a large file via mail |

### Check Wazuh is reading mail logs

```bash
# On the node running Wazuh agent
cat /var/ossec/etc/ossec.conf | grep mail
```

Add to `ossec.conf` if not present:

```xml
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/mail.log</location>
</localfile>
```

---

## 📁 Key File Locations

| File | Purpose |
|---|---|
| `/etc/postfix/main.cf` | Main Postfix config |
| `/etc/postfix/master.cf` | Postfix service/daemon config |
| `/etc/postfix/transport` | Manual routing rules |
| `/var/spool/postfix/etc/hosts` | Hosts file inside Postfix chroot |
| `/var/log/mail.log` | Mail logs (sender and receiver) |
| `/home/mailuser/Maildir/new/` | Incoming emails for mailuser |

---

## 🛠️ Useful Commands

```bash
# Check mail queue
mailq

# Check Postfix config values
postconf myhostname mydestination mynetworks

# Watch logs live
tail -f /var/log/mail.log

# Check Postfix is running
systemctl status postfix

# Check port 25 is open
ss -tlnp | grep :25

# Test connection to remote mail server
telnet <NODE-B-IP> 25

# Send test email
echo "Test body" | mail -s "Test Subject" mailuser@nodeb.local

# Read received email
cat /home/mailuser/Maildir/new/<filename>
```

---

## 📚 References

- [Postfix Documentation](http://www.postfix.org/documentation.html)
- [Wazuh Log Analysis](https://documentation.wazuh.com/current/user-manual/capabilities/log-data-collection/index.html)
- [Ubuntu 24.04 Mail Setup](https://ubuntu.com/server/docs/mail-postfix)
- [Maildir Format](https://en.wikipedia.org/wiki/Maildir)

---

## 👤 Author

Built during a hands-on lab session for learning mail server log analysis with Wazuh SIEM integration.

> Replace `<NODE-A-IP>`, `<NODE-B-IP>`, and `<YOUR-NETWORK-CIDR>` with your actual values before using this guide.
