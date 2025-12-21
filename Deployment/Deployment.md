### Infrastructure Requirements & Deployment Notes

#### Core Dependencies
| Component                   | Version           | CPU      | RAM       | Disk Type | Disk Space | Notes |
|----------------------------|-------------------|----------|-----------|-----------|------------|------|
| ElasticSearch / OpenSearch | ≥ 8.0 / ≥ 2.9     | 2 cores  | ≥ 8 GB    | SSD       | ≥ 16 GB    | Primary datastore – performance critical |
| Redis                      | ≥ 7.1             | 1 core   | ≥ 1 GB    | SSD       | ≥ 16 GB    | Cache & queue optimization |
| RabbitMQ                   | ≥ 3.11            | 1 core   | ≥ 512 MB  | Standard  | ≥ 2 GB     | Message broker for workers |
| S3 / MinIO                 | ≥ RELEASE.2023-02 | 1 core   | ≥ 128 MB  | SSD       | ≥ 16 GB    | File storage (imports, exports, reports) |

> **Recommendation:**  
> ElasticSearch/OpenSearch **must run on SSD storage** to avoid indexing delays and ingestion latency.

#### OpenCTI Platform Components
All OpenCTI application services are **stateless** and can be scaled horizontally if needed.
| Component      | CPU      | RAM       | Disk Type | Notes |
|---------------|----------|-----------|-----------|------|
| OpenCTI Core  | 2 cores  | ≥ 8 GB    | None      | API & Web UI backend |
| Worker(s)     | 1 core   | ≥ 128 MB  | None      | Background processing |
| Connector(s)  | 1 core   | ≥ 128 MB  | None      | External data ingestion |
| XTM Composer  | 1 core   | ≥ 128 MB  | None      | Threat modeling component |

#### Minimum System Requirements (Production)
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

### System Preparation & Docker Installation

#### Install Docker Engine & Required Plugins
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
#### Install Supporting Tools
```
dnf install jq git -y
```
#### System Tuning (Required)
ElasticSearch Kernel Parameter
```
sudo sysctl -w vm.max_map_count=1048575
echo 'vm.max_map_count=1048575' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```
#### Disable Firewall
```
systemctl stop firewalld
systemctl disable firewalld
```
#### Clone OpenCTI Docker Repository
```
mkdir ~/opencti && cd ~/opencti
git clone https://github.com/OpenCTI-Platform/docker.git
cd docker
```
#### Configure Environment Variables
```
cp .env.sample .env
chmod 600 .env
nano .env
```
#### UUIDv4 Generator
Use a UUIDv4 generator when required (tokens, IDs, secrets): [uuidgenerator](https://www.uuidgenerator.net/)

#### Start OpenCTI
```
docker compose pull 
docker compose up -d
```
#### Production Deployment (Post-PoC)
Docker Swarm (Recommended for Scaling)
On the manager node:
```
docker swarm init
docker stack deploy -c docker-compose.yml opencti
```
#### Validation & Troubleshooting
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
docker logs -f opencti
```
> Use container-specific logs for deeper troubleshooting when required.
