# Production Web Infrastructure & Secure Reverse Proxy Deployment

## 📌 Project Overview
This project demonstrates the deployment and hardening of a multi-tier web infrastructure on **AWS EC2 (Ubuntu Linux)**. 

To achieve optimal performance, security isolation, and attack surface reduction:
* **Nginx** operates as the public-facing **Reverse Proxy** on port `80`.
* **Apache HTTP Server** is isolated on the loopback interface (`127.0.0.1:8080`) as the internal backend application server.
* System-level firewall policies are enforced using **UFW (Uncomplicated Firewall)** to strictly restrict unauthorized network access.

---

## 🏗️ Architecture & Traffic Flow

```text
[ Client / Public Web Browser ]
              |
              | (HTTP - Port 80)
              v
[ Host-Level UFW Firewall ] (Allows 22, 80, 443)
              |
              v
[ Nginx Reverse Proxy (Frontend) ] (Listens on 0.0.0.0:80)
              |
              | (Internal Loopback Proxy Forwarding)
              v
[ Apache HTTP Server (Backend) ] (Isolated on 127.0.0.1:8080)
```

---

## 🛠️ Step-by-Step Implementation

### Step 1: Package Updates & Apache Backend Setup
1. Update system package repositories:
   ```bash
   sudo apt update && sudo apt upgrade -y
   ```

2. Install Apache web server:
   ```bash
   sudo apt install apache2 -y
   ```

3. Configure Apache to bind strictly to localhost on port `8080`:
   * Edit `/etc/apache2/ports.conf`:
     ```apache
     Listen 127.0.0.1:8080
     ```
   * Edit VirtualHost definition in `/etc/apache2/sites-available/000-default.conf`:
     ```apache
     <VirtualHost 127.0.0.1:8080>
         ServerAdmin webmaster@localhost
         DocumentRoot /var/www/html
         ErrorLog ${APACHE_LOG_DIR}/error.log
         CustomLog ${APACHE_LOG_DIR}/access.log combined
     </VirtualHost>
     ```

4. Create a sample backend response page:
   ```bash
   echo "<h1>Hello! Response from Backend Apache running on Port 8080</h1>" | sudo tee /var/www/html/index.html
   ```

5. Restart and enable Apache:
   ```bash
   sudo systemctl restart apache2
   sudo systemctl enable apache2
   ```

---

### Step 2: Nginx Reverse Proxy Configuration
1. Install Nginx:
   ```bash
   sudo apt install nginx -y
   ```

2. Remove the default virtual host configuration:
   ```bash
   sudo rm /etc/nginx/sites-enabled/default
   ```

3. Create the reverse proxy configuration at `/etc/nginx/sites-available/reverse-proxy.conf`:
   ```nginx
   server {
       listen 80;
       server_name _;

       location / {
           proxy_pass [http://127.0.0.1:8080](http://127.0.0.1:8080);
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;
       }
   }
   ```

4. Enable the configuration and verify syntax:
   ```bash
   sudo ln -s /etc/nginx/sites-available/reverse-proxy.conf /etc/nginx/sites-enabled/
   sudo nginx -t
   sudo systemctl restart nginx
   sudo systemctl enable nginx
   ```

---

### Step 3: Host Firewall Hardening (UFW)
Secure the host by strictly permitting only operational ports (SSH, HTTP, HTTPS) while keeping backend port `8080` isolated:

```bash
# Allow essential administration and web traffic
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Enable firewall and verify rules
sudo ufw enable
sudo ufw status verbose
```

---

## 🔍 Verification & Diagnostics

### 1. Port Binding Verification
Ensure Apache is restricted to `127.0.0.1:8080` and Nginx is bound to public port `80`:
```bash
sudo ss -tulpn | grep -E 'apache2|nginx'
```

### 2. Live Access Log Monitoring
Track incoming real-time requests and client IP forwarding:
```bash
sudo tail -f /var/log/nginx/access.log
```

### 3. Outage Simulation (502 Bad Gateway Troubleshooting)
* Simulate an upstream backend failure:
  ```bash
  sudo systemctl stop apache2
  ```
* Accessing the public IP triggers a **`502 Bad Gateway`**.
* Inspect upstream connection failure logs:
  ```bash
  sudo tail -n 5 /var/log/nginx/error.log
  ```
  *Output confirms:* `connect() failed (111: Connection refused) while connecting to upstream: "http://127.0.0.1:8080/"`
* Restore service availability:
  ```bash
  sudo systemctl start apache2
  ```

---

## 🚀 Key Takeaways & Sysadmin Skills Demonstrated
* **Reverse Proxy Architecture:** Reduced backend server exposure while handling request routing efficiently.
* **Header Preservation:** Correctly forwarded original client IP addresses (`X-Real-IP`, `X-Forwarded-For`) to backend web servers.
* **Firewall Hardening:** Restricted external attack vectors via host-level packet filtering with UFW.
* **Production Troubleshooting:** Diagnosed upstream proxy connection failures using standard Linux log pipelines (`/var/log/nginx/`).
