Blocking Facebook on an Ubuntu server can be achieved by modifying its **DNS settings**, **IP routing**, or using a **firewall** like `iptables` or `ufw`. Here's a complete guide covering different methods:

---

### **Method 1: Block Facebook Using `/etc/hosts`**

1. **Edit the `/etc/hosts` file**:
   ```bash
   sudo nano /etc/hosts
   ```

2. **Add Facebook domains to the file**:
   Redirect Facebook domains to `127.0.0.1` (localhost):
   ```plaintext
   127.0.0.1 facebook.com
   127.0.0.1 www.facebook.com
   127.0.0.1 m.facebook.com
   127.0.0.1 graph.facebook.com
   127.0.0.1 static.facebook.com
   127.0.0.1 fbcdn.net
   127.0.0.1 www.fbcdn.net
   127.0.0.1 fbcdn.com
   127.0.0.1 www.fbcdn.com
   ```

3. **Flush DNS Cache**:
   If caching is enabled, flush the DNS cache:
   ```bash
   sudo systemctl restart systemd-resolved
   ```

4. **Verify Blocking**:
   Try pinging Facebook:
   ```bash
   ping facebook.com
   ```
   It should resolve to `127.0.0.1`.

---

### **Method 2: Block Facebook Using `iptables`**

1. **Find Facebook IPs**:
   Run this command to get Facebook's IP addresses:
   ```bash
   nslookup facebook.com
   ```
   (Or check tools like [whois.domaintools.com](https://whois.domaintools.com) for IP blocks).

2. **Add Firewall Rules**:
   Use `iptables` to block IP ranges:
   ```bash
   sudo iptables -A OUTPUT -p tcp -d 157.240.0.0/16 -j REJECT
   sudo iptables -A OUTPUT -p udp -d 157.240.0.0/16 -j REJECT
   ```

3. **Persist Firewall Rules**:
   Save the rules so they persist after reboot:
   ```bash
   sudo iptables-save > /etc/iptables/rules.v4
   ```

4. **Verify Blocking**:
   Test by accessing Facebook using `curl`:
   ```bash
   curl facebook.com
   ```

---

### **Method 3: Block Facebook Using `ufw`**

1. **Enable `ufw`**:
   If `ufw` is not enabled, activate it:
   ```bash
   sudo ufw enable
   ```

2. **Block Facebook's IPs**:
   Deny access to Facebook's IP ranges:
   ```bash
   sudo ufw deny out to 157.240.0.0/16
   ```

3. **Check Rules**:
   Verify the rules:
   ```bash
   sudo ufw status
   ```

---

### **Method 4: Use a Proxy Server with Filtering**

For complex networks, a proxy server like **Squid** or a filtering tool like **SquidGuard** or **DansGuardian** can block websites.

#### **Setup Squid Proxy**

1. **Install Squid**:
   ```bash
   sudo apt update
   sudo apt install squid
   ```

2. **Configure Squid**:
   Edit the Squid configuration file:
   ```bash
   sudo nano /etc/squid/squid.conf
   ```

   Add these lines to block Facebook:
   ```plaintext
   acl blocksites dstdomain .facebook.com
   http_access deny blocksites
   ```

3. **Restart Squid**:
   ```bash
   sudo systemctl restart squid
   ```

4. **Verify Blocking**:
   Ensure your traffic passes through Squid to see the effect.

---

### **Method 5: Block Facebook Using DNS**

#### **Using `/etc/resolv.conf`**
You can redirect DNS queries for Facebook to a non-existent server:

1. **Edit DNS Resolver**:
   ```bash
   sudo nano /etc/resolv.conf
   ```

   Add entries pointing to an invalid DNS:
   ```plaintext
   nameserver 127.0.0.1
   ```

2. **Set Up a Fake DNS**:
   Use a DNS resolver like `dnsmasq` to resolve Facebook domains to localhost.

#### **Using Pi-hole** (Network-wide blocking)
Set up **Pi-hole** as a DNS server to block Facebook across all devices on your network.

1. **Install Pi-hole**:
   Follow installation steps from the [official Pi-hole site](https://pi-hole.net/).

2. **Add Facebook to Blocklist**:
   Include Facebook in the Pi-hole blocklist.

---

### **Additional Tips**

- To ensure comprehensive blocking, include Facebook subdomains like `apps.facebook.com`, `connect.facebook.net`, and `fb.me`.
- Consider combining multiple methods for robust blocking.
- Remember to update IP ranges periodically, as services like Facebook may change their infrastructure.

Let me know if you need help with any specific step!
