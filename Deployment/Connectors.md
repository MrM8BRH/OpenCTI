### Adding OpenCTI Connectors

#### Clone the Official Connectors Repository
```
git clone https://github.com/OpenCTI-Platform/connectors.git
cd connectors
```
> Note:
>
> It is recommended to keep connectors separate from the OpenCTI core directory for better lifecycle management and scaling.

#### Select the Required Connector(s)
Navigate to the connector directory you wish to deploy, for example:
- `external-import/misp`
- `external-import/mitre`

> Best Practice:
> 
> Deploy only required connectors to reduce resource usage and noise in OpenCTI.

#### Configure Connector Environment Variables
Modify one of the following files:
- `docker-compose.yml` 
- `.env`
```
# Unique identifier for this connector instance (UUIDv4)
CONNECTOR_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

# OpenCTI URL
OPENCTI_URL=http://opencti:8080

# OpenCTI API token
OPENCTI_TOKEN=xxx-your-api-token-xxx
```
> Get API token from OpenCTI UI → Profile → API access

#### Start the connector:
```
docker compose up -d
```
> Repeat for each connector you need.
