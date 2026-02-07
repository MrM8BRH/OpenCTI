### Infrastructure Requirements & Deployment Notes

<details>
<summary><b>Core Dependencies</b></summary>

| Component                   | Version           | CPU      | RAM       | Disk Type | Disk Space | Notes |
|----------------------------|-------------------|----------|-----------|-----------|------------|------|
| ElasticSearch / OpenSearch | ≥ 8.0 / ≥ 2.9     | 2 cores  | ≥ 8 GB    | SSD       | ≥ 16 GB    | Primary datastore – performance critical |
| Redis                      | ≥ 7.1             | 1 core   | ≥ 1 GB    | SSD       | ≥ 16 GB    | Cache & queue optimization |
| RabbitMQ                   | ≥ 3.11            | 1 core   | ≥ 512 MB  | Standard  | ≥ 2 GB     | Message broker for workers |
| S3 / MinIO                 | ≥ RELEASE.2023-02 | 1 core   | ≥ 128 MB  | SSD       | ≥ 16 GB    | File storage (imports, exports, reports) |

> **Recommendation:**  
> ElasticSearch/OpenSearch **must run on SSD storage** to avoid indexing delays and ingestion latency.
</details>

<details>
<summary><b>OpenCTI Platform Components</b></summary>
  
All OpenCTI application services are **stateless** and can be scaled horizontally if needed.
| Component      | CPU      | RAM       | Disk Type | Notes |
|---------------|----------|-----------|-----------|------|
| OpenCTI Core  | 2 cores  | ≥ 8 GB    | None      | API & Web UI backend |
| Worker(s)     | 1 core   | ≥ 128 MB  | None      | Background processing |
| Connector(s)  | 1 core   | ≥ 128 MB  | None      | External data ingestion |
| XTM Composer  | 1 core   | ≥ 128 MB  | None      | Threat modeling component |
</details>

<details>
<summary><b>Minimum System Requirements (Production)</b></summary>

Recommended baseline for **production environments**:
| Component      | Recommended                         |
| -------------- | ----------------------------------- |
| CPU            | 16+ vCPUs                            |
| RAM            | 32+ GB                              |
| Disk           | SSD, 600 GB+                        |
| Docker         | 20+                                 |
| Docker Compose | v2.x                                |
> **Note:**  
> Larger datasets, multiple connectors, or high ingestion rates require additional CPU and RAM, especially for ElasticSearch/OpenSearch.
</details>

<details>
<summary><b>Preparing a System Before OpenCTI Installation</b></summary>
  
<details>
<summary><b>Update the system & Install additional tools</b></summary>

RHEL family
```
yum update -y
yum install -y dnf
dnf install -y net-tools nano bind-utils chkconfig wget net-tools cloud-utils-growpart
```
Debian family
```
apt update -y
apt full-upgrade -y
apt install -y net-tools nano wget net-tools cloud-guest-utils
```
</details>

<details>
<summary><b>Disable Firewall</b></summary>

RHEL family
```
systemctl stop firewalld
systemctl disable firewalld
```
Debian family
```
systemctl stop ufw
systemctl disable ufw
```
</details>


<details>
<summary><b>Disable Transparent Huge Pages (THP)</b></summary>

*   `nano /etc/systemd/system/disable-thp.service`
```
[Unit]
Description=Disable Transparent Huge Pages (THP)

[Service]
Type=simple
ExecStart=/bin/sh -c "echo 'never' > /sys/kernel/mm/transparent_hugepage/enabled && echo 'never' > /sys/kernel/mm/transparent_hugepage/defrag"

[Install]
WantedBy=multi-user.target
```

```
systemctl daemon-reload
systemctl start disable-thp
systemctl enable disable-thp
```
</details>

<details>
<summary><b>System Tuning (Required)</b></summary>

Add the following kernel settings to your `/etc/sysctl.conf` file:
ElasticSearch Kernel Parameter
```
sudo sysctl -w vm.max_map_count=1048575
echo 'vm.max_map_count=1048575' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```
</details>


```diff
- After completing the above, restart the system
reboot
```
</details>

<details>
<summary><b>Docker Installation</b></summary>

#### Install Docker Engine (RHEL)
```
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
#### Verify Docker Compose Version
```
docker compose version
```
#### Start & Enable Docker Service
```
systemctl start docker.socket
systemctl enable docker.socket
```
</details>

<details>
<summary><b>OpenCTI Installation & Configration</b></summary>

#### Download OpenCTI Docker Compose Files
```
mkdir -p /opt/opencti && cd /opt/opencti
wget https://raw.githubusercontent.com/OpenCTI-Platform/docker/refs/heads/master/docker-compose.yml
wget https://raw.githubusercontent.com/OpenCTI-Platform/docker/refs/heads/master/rabbitmq.conf
wget https://raw.githubusercontent.com/OpenCTI-Platform/docker/refs/heads/master/.env.sample
```
#### Configure Environment Variables
```
cp .env.sample .env
nano .env
```
Change the following
```
OPENCTI_ADMIN_PASSWORD=changeme
OPENCTI_ADMIN_TOKEN=ChangeMe_UUIDv4
OPENCTI_HEALTHCHECK_ACCESS_KEY=changeme
```
#### Start OpenCTI
```
docker compose up -d
```
</details>

<details>
<summary><b>Add Cerificate to OpenCTI</b></summary>

<details>
<summary><b>Using OpenCTI Containers</b></summary>

[Configuration keys](https://docs.opencti.io/6.9.X/deployment/configuration/)
```
mkdir -p /opt/opencti/certs
cd /opt/opencti/certs
# key.pem cert.pem ca.pem
```
.env
```
OPENCTI_HEALTHCHECK_ACCESS_KEY=changeme
OPENCTI_EXTERNAL_SCHEME=http
```
docker-compose.yml
```
  opencti:
    image: opencti/platform:6.9.10
    environment:
      - APP__HTTPS_CERT__KEY=/certs/key.pem
      - APP__HTTPS_CERT__CRT=/certs/cert.pem
      - APP__HTTPS_CERT__CA=["/certs/ca.pem"]
      - APP__HTTPS_CERT__REJECT_UNAUTHORIZED=true
      - APP__HTTPS_CERT__COOKIE_SECURE=true
    volumes:
      - /opt/opencti/certs:/certs:ro
    healthcheck:
      test:  ["CMD-SHELL", "wget --no-check-certificate -qO- \"https://localhost:8080/health?health_access_key=$(printenv APP__HEALTH_ACCESS_KEY)\" || exit 1"]
      interval: 20s
      timeout: 10s
      retries: 30
```
</details>

<details>
<summary><b>RHEL (Nginx)</b></summary>

### Step 1: Install Nginx
```
dnf install nginx -y
```
### Step 2: Configure SSL Certificates

Configure SSL Certificates
```
mv key.pem /etc/pki/tls/private/key.pem
mv cert.pem /etc/pki/tls/certs/cert.pem
```
Fix permissions
```
chown root:root /etc/pki/tls/private/key.pem
chown root:root /etc/pki/tls/certs/cert.pem
chmod 600 /etc/pki/tls/private/key.pem
chmod 644 /etc/pki/tls/certs/cert.pem
```
### Step 3: Create an Nginx Configuration
```
nano /etc/nginx/conf.d/opencti.conf
```
Add the following configuration to the file:
```
server {
listen 443 ssl;
server_name your-domain.com;

ssl_certificate /etc/pki/tls/certs/cert.pem;
ssl_certificate_key /etc/pki/tls/private/key.pem;

location / {
    proxy_pass http://localhost:8080;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
	}
}

server {
listen 80;
server_name your-domain.com;
return 301 https://$host$request_uri;
}
```
```
# This permanently allows Nginx `(httpd_t)` to connect to backend services (like Docker).
setsebool -P httpd_can_network_connect on
```

### Step 4: Test Nginx Configuration and Restart
Test the Nginx configuration for syntax errors:
```
sudo nginx -t
```

If the test is successful, restart Nginx to apply the changes:
```
systemctl restart nginx
```

### Step 5: Update OpenCTI Configuration
Update the APP__BASE_URL environment variable in your OpenCTI service configuration to use https:
- `APP__BASE_URL=https://your-domain.com`

After completing these steps, Nginx will handle SSL termination for your OpenCTI service and proxy requests to `http://localhost:8080` internally. Access your OpenCTI service securely via `https://your-domain.com`.
</details>

<details>
<summary><b>Debian (Nginx)</b></summary>

### Step 1: Install Nginx
```
apt install nginx -y
```
### Step 2: Configure SSL Certificates

```
mkdir -p /etc/ssl/opencti
cp cert.pem key.pem /etc/ssl/opencti/
```
Fix permissions
```
chown root:root /etc/ssl/opencti/key.pem
chown root:root /etc/ssl/opencti/cert.pem
chmod 600 /etc/ssl/opencti/key.pem
chmod 644 /etc/ssl/opencti/cert.pem
```
### Step 3: Create an Nginx Configuration
```
nano /etc/nginx/sites-available/opencti
```
Add the following configuration to the file:
```
server {
listen 443 ssl;
server_name your-domain.com;

ssl_certificate /etc/ssl/opencti/cert.pem;
ssl_certificate_key /etc/ssl/opencti/key.pem;

location / {
    proxy_pass http://localhost:8080;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
	}
}

server {
listen 80;
server_name your-domain.com;
return 301 https://$host$request_uri;
}
```

### Step 4: Enable the Nginx Configuration
Create a symbolic link to enable the new configuration:
```
sudo ln -s /etc/nginx/sites-available/opencti /etc/nginx/sites-enabled/
```
### Step 5: Test Nginx Configuration and Restart
Test the Nginx configuration for syntax errors:
```
sudo nginx -t
```

If the test is successful, restart Nginx to apply the changes:
```
systemctl restart nginx
```

### Step 6: Update OpenCTI Configuration
Update the APP__BASE_URL environment variable in your OpenCTI service configuration to use https:
- `APP__BASE_URL=https://your-domain.com`

After completing these steps, Nginx will handle SSL termination for your OpenCTI service and proxy requests to `http://localhost:8080` internally. Access your OpenCTI service securely via `https://your-domain.com`.
</details>

</details>

<details>
<summary><b>Configure a Live Stream Between Two OpenCTI Instances</b></summary>

### 1) OpenCTI_A (Source) – Sharing User & Token
- Create a dedicated user `Settings → Security → Users` → e.g. `[S] OpenCTI_B Stream`
- Add the user to the `Connectors` group
- Grant `Access data sharing` permission - *(Optional: `Manage data sharing` if the user will manage streams)*
- Copy the user **API token** (used by OpenCTI_B)

### 2) OpenCTI_A (Source) – Create Live Stream
- Path: `Data → Data sharing → Live streams → CREATE`
- **Name**: `Sync to OpenCTI_B - Indicators`
- **Filters** (example):
  - Entity type = `Indicator`
  - Optional: `pattern type`, `labels`, `markings`, `confidence`
- Save and start the stream
- **Note**: Dependencies/relationships may also be sent automatically
- **Stream endpoint**:
  ```
  https://opencti_a.example/stream/{STREAM_ID}
  ```
### 3) OpenCTI_B (Destination)
- Path: `Data → Ingestion → OpenCTI Streams`
- **Remote OpenCTI URL**: *(Base URL only)*
  ```
  https://opencti-a.example
  ```
- **Remote token**: OpenCTI_A sharing user token
- Validate the connection
- Select the Live Stream created on OpenCTI_A

#### Important Options
- **User responsible for data creation**: Choose a dedicated “import” user on OpenCTI_B for traceability.
- **Starting synchronization**: Set the oldest date to pull.
- **Take deletions into account**: If enabled, deletions on OpenCTI_A can delete on OpenCTI_B
- **Verify SSL certificate**: Enable for production; disable only for lab/self-signed testing
- **Avoid dependencies resolution**: Imports only entities without relationships (useful if you want minimal objects)
- **Use perfect synchronization**: Imported objects overwrite existing ones (true “sync” behavior)
- Save and start the stream

</details>

<details>
<summary><b>UUIDv4 Generator</b></summary>

Use a UUIDv4 generator when required (tokens, IDs, secrets): [uuidgenerator](https://www.uuidgenerator.net/)
</details>

<details>
<summary><b>Validation & Troubleshooting</b></summary>

- Verify Running Containers (Expect approximately ~20 containers)
```
docker ps -a
```
- Validate Web UI Availability (Expected response: HTTP/1.1 200 OK)
```
curl -I http://opencti:8080
```
- View Logs
```
docker logs container_name
```
> Use container-specific logs for deeper troubleshooting when required.
</details>

<details>
<summary><b>Resources</b></summary>

- [Top 100 Cyber Security RSS Feeds (Security RSS)](https://rss.feedspot.com/cyber_security_rss_feeds/)
</details>
