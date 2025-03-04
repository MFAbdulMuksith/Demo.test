### **How to Run Promtail and Loki on Your Servers for Log Monitoring?**  

Since you already have **Prometheus, Grafana, and Loki** in your **docker-compose.yml**, you now need to:  
1️⃣ **Deploy Loki** (already in your compose, just ensure it runs correctly).  
2️⃣ **Deploy Promtail** on each server (`Achieve-Cargo-Prod-1-V2` & `Achieve-Cargo-Prod-2-V2`).  

---

## **1️⃣ Deploy Loki on Your Monitoring Server**
🔹 Loki is already in your `docker-compose.yml`, but let's confirm and **run it properly.**  

### **✅ Verify Loki Configuration in `docker-compose.yml`**
Your Loki config should look like this:
```yaml
  loki:
    image: grafana/loki:3.2.1
    restart: always
    container_name: loki
    ports:
      - 3100:3100  # Loki API port
    command:
      - '--config.file=/etc/loki/loki-config.yaml'  # Standard path for config files in containers
    volumes:
      - /opt/prometheus/loki-config.yaml:/etc/loki/loki-config.yaml  # Correct mapping path for config
      - /opt/prometheus/loki-data:/data/loki  # Persistent data directory
    networks:
      - monitor  # Loki connected to the 'monitor' network
```
👉 **If it's already in your config, just start Loki**:  
```bash
docker-compose up -d loki
```
👉 **Check if Loki is running correctly**:  
```bash
docker logs -f loki
```
👉 **Verify Loki API is accessible**:  
```bash
curl http://localhost:3100/ready
```
If Loki is running fine, it should return:  
```json
"ready"
```

---

## **2️⃣ Deploy Promtail on Each Server (`10.0.2.53` & `10.0.3.194`)**  

🔹 Promtail collects logs from your servers and **sends them to Loki**.  
🔹 Since your servers are not running Docker, you'll **install Promtail manually** on each.

### **✅ Step 1: Download Promtail on Each Server**
Run these commands on both servers (`10.0.2.53` & `10.0.3.194`):

```bash
cd /opt/
wget https://github.com/grafana/loki/releases/latest/download/promtail-linux-amd64 -O promtail
chmod +x promtail
```
This downloads Promtail and makes it executable.

---

### **✅ Step 2: Create Promtail Config File (`/opt/promtail-config.yaml`)**
Create the config file on **each server** (`10.0.2.53` & `10.0.3.194`):

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /opt/promtail-positions.yaml  # Stores positions of already-read logs

clients:
  - url: "http://YOUR-LOKI-SERVER:3100/loki/api/v1/push"  # Change to your Loki server

scrape_configs:
  - job_name: "varlogs"
    static_configs:
      - targets:
          - localhost
        labels:
          job: "varlogs"
          host: "server-1"  # Change this for each server
          __path__: "/var/log/*.log"
```

🔹 **Make sure to change:**  
- `http://YOUR-LOKI-SERVER:3100` → Replace with the actual Loki server **IP or Hostname**  
- `host: "server-1"` → Change to `server-2` for the second server  
- `__path__: "/var/log/*.log"` → Monitors `/var/log/` logs. Change if needed.

---

### **✅ Step 3: Run Promtail as a Background Service**
Now, start Promtail on each server:

```bash
nohup /opt/promtail -config.file /opt/promtail-config.yaml > /dev/null 2>&1 &
```
This runs Promtail in the background.

---

### **✅ Step 4: Verify Promtail is Running**
Run:
```bash
ps aux | grep promtail
```
If running, it should return a process similar to:
```
root      12345  0.1  0.2  123456  67890 ?  Ssl  14:32   0:02 /opt/promtail -config.file /opt/promtail-config.yaml
```

---

## **3️⃣ Configure Grafana to Use Loki for Logs**
1. Open Grafana (`http://localhost:3000`).  
2. Go to **Settings** → **Data Sources** → **Add Data Source**.  
3. Choose **Loki** and enter the Loki URL:
   ```
   http://YOUR-LOKI-SERVER:3100
   ```
4. Click **Save & Test** → Should show "Data source is working!".  

---

## **4️⃣ Test Logs in Grafana**
1. Go to **Explore** → Choose **Loki** as the data source.  
2. In the query field, enter:  
   ```
   {job="varlogs"}
   ```
3. Click **Run Query** → You should see logs from `/var/log/*.log` from both servers. 🎉  

---

## **✅ Summary**
| Component | Installed On | Status |
|-----------|------------|--------|
| **Loki**  | **Monitoring Server** (Prometheus) | ✅ Running on `3100` |
| **Promtail** | **Each Server (10.0.2.53 & 10.0.3.194)** | ✅ Sends logs to Loki |
| **Grafana** | **Monitoring Server** | ✅ Reads logs from Loki |

🚀 **You now have logs from both servers being monitored in Grafana!**


---

# ======================================


---


To run **Promtail** and **Loki** containers for log aggregation and monitoring on your servers, follow these steps. This setup will allow you to collect logs from your servers and forward them to Loki, where they can be queried and visualized in Grafana.

---

### **1. Prerequisites**
- Docker and Docker Compose are installed on your servers.
- Access to the servers where you want to collect logs.
- A running Grafana instance (already part of your `docker-compose.yaml`).

---

### **2. Loki Configuration**

#### **2.1 Create Loki Configuration File**
Create a `loki-config.yaml` file to configure Loki. Save it at `/opt/prometheus/loki-config.yaml` (as referenced in your `docker-compose.yaml`).

```yaml
auth_enabled: false

server:
  http_listen_port: 3100

common:
  path_prefix: /data/loki
  storage:
    filesystem:
      chunks_directory: /data/loki/chunks
      rules_directory: /data/loki/rules
  replication_factor: 1
  ring:
    instance_addr: 127.0.0.1
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

ruler:
  alertmanager_url: http://alertmanager:9093
```

---

### **3. Promtail Configuration**

#### **3.1 Create Promtail Configuration File**
Create a `promtail-config.yaml` file to configure Promtail. Save it at `/opt/prometheus/promtail-config.yaml`.

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /var/lib/promtail/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: system
    static_configs:
      - targets:
          - localhost
        labels:
          job: varlogs
          __path__: /var/log/*.log
```

This configuration will collect logs from `/var/log/*.log` on the server where Promtail is running.

---

### **4. Update Docker Compose**

Add the **Loki** and **Promtail** services to your `docker-compose.yaml` file. Ensure they are connected to the `monitor` network.

```yaml
version: '3.9'

services:
  # Existing services (Prometheus, Grafana, etc.)
  ...

  # Loki Configuration
  loki:
    image: grafana/loki:3.2.1
    restart: always
    container_name: loki
    ports:
      - 3100:3100  # Loki API port
    volumes:
      - /opt/prometheus/loki-config.yaml:/etc/loki/loki-config.yaml
      - /opt/prometheus/loki-data:/data/loki
    command:
      - '--config.file=/etc/loki/loki-config.yaml'
    networks:
      - monitor

  # Promtail Configuration
  promtail:
    image: grafana/promtail:3.2.1
    restart: always
    container_name: promtail
    volumes:
      - /opt/prometheus/promtail-config.yaml:/etc/promtail/promtail-config.yaml
      - /var/log:/var/log  # Mount the host's log directory
      - /opt/prometheus/promtail-positions:/var/lib/promtail
    command:
      - '--config.file=/etc/promtail/promtail-config.yaml'
    networks:
      - monitor

networks:
  monitor:
    external: true
```

---

### **5. Start the Services**

1. **Create the `monitor` Network** (if it doesn’t exist):
   ```bash
   docker network create monitor
   ```

2. **Start the Services**:
   ```bash
   docker-compose up -d
   ```

---

### **6. Verify Loki and Promtail**

1. **Check Loki**:
   - Open `http://localhost:3100/ready` in your browser. It should return `ready`.

2. **Check Promtail**:
   - Check the logs of the Promtail container to ensure it is running and forwarding logs to Loki:
     ```bash
     docker logs promtail
     ```

3. **Verify Logs in Loki**:
   - Use the Loki API to query logs:
     ```bash
     curl -G http://localhost:3100/loki/api/v1/query --data-urlencode 'query={job="varlogs"}'
     ```

---

### **7. Configure Grafana to Use Loki**

1. **Add Loki as a Data Source**:
   - Go to `http://localhost:3000` (Grafana).
   - Navigate to **Configuration > Data Sources > Add data source**.
   - Select **Loki**.
   - Set the URL to `http://loki:3100`.
   - Click **Save & Test**.

2. **Create a Dashboard**:
   - Go to **Explore** in Grafana.
   - Select the Loki data source.
   - Use the query `{job="varlogs"}` to view logs.

---

### **8. Deploy Promtail to Other Servers**

If you want to collect logs from multiple servers, deploy Promtail to each server:

1. **Create a Promtail Configuration File** for each server:
   - Update the `clients` section to point to your Loki instance (e.g., `http://<loki-server-ip>:3100/loki/api/v1/push`).

2. **Run Promtail on Each Server**:
   Use Docker to run Promtail on each server:

   ```bash
   docker run -d \
     --name promtail \
     -v /opt/prometheus/promtail-config.yaml:/etc/promtail/promtail-config.yaml \
     -v /var/log:/var/log \
     -v /opt/prometheus/promtail-positions:/var/lib/promtail \
     grafana/promtail:3.2.1 \
     --config.file=/etc/promtail/promtail-config.yaml
   ```

---

### **9. Query Logs in Grafana**

- Use the **Explore** feature in Grafana to query logs from Loki.
- Example query: `{job="varlogs"}` to view logs from all servers.

---

### **Summary**

- **Loki** is the log aggregation system.
- **Promtail** collects logs from your servers and forwards them to Loki.
- **Grafana** is used to visualize and query logs.

This setup provides a centralized logging solution for your servers. Let me know if you need further assistance!
