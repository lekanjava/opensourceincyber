---
layout: page
title: Docker Deployment Guide
permalink: /tutorials/docker-deploy/
---

# Docker Deployment Guide for Security Tools

This tutorial provides step-by-step instructions for deploying common security tools using Docker, making it easier to set up and maintain your security infrastructure.

## Prerequisites

Before you begin, ensure you have the following installed:

- Docker Engine (version 20.10.x or later)
- Docker Compose (version 2.x or later)
- Basic understanding of Docker concepts
- At least 4GB of RAM and 20GB of free disk space

## Setting Up Docker Environment

First, let's ensure Docker is properly configured for security:

```bash
# Create a dedicated network for security tools
docker network create security-tools-network

# Set up a volume for persistent data
docker volume create security-data
```

## Deploying SIEM: Wazuh

Wazuh is a free, open-source security monitoring solution that provides threat detection, integrity monitoring, and compliance capabilities.

### Step 1: Create a docker-compose.yml file

```yaml
version: '3.9'

services:
  wazuh:
    image: wazuh/wazuh:4.3.10
    hostname: wazuh-manager
    restart: always
    ports:
      - "1514:1514"
      - "1515:1515"
      - "514:514/udp"
      - "55000:55000"
    volumes:
      - wazuh-config:/var/ossec/etc
      - wazuh-logs:/var/ossec/logs
      - wazuh-queue:/var/ossec/queue
      - wazuh-var:/var/ossec/var
      - wazuh-integrations:/var/ossec/integrations
      - wazuh-active-response:/var/ossec/active-response
      - wazuh-agentless:/var/ossec/agentless
      - wazuh-wodles:/var/ossec/wodles
      - filebeat-etc:/etc/filebeat
      - filebeat-var:/var/lib/filebeat
    networks:
      - security-tools-network

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.17.0
    restart: always
    environment:
      - "ES_JAVA_OPTS=-Xms1g -Xmx1g"
      - "bootstrap.memory_lock=true"
      - "discovery.type=single-node"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - elastic-data:/usr/share/elasticsearch/data
    networks:
      - security-tools-network

  kibana:
    image: docker.elastic.co/kibana/kibana:7.17.0
    restart: always
    depends_on:
      - elasticsearch
    ports:
      - "5601:5601"
    environment:
      - "ELASTICSEARCH_HOSTS=http://elasticsearch:9200"
    networks:
      - security-tools-network

volumes:
  wazuh-config:
  wazuh-logs:
  wazuh-queue:
  wazuh-var:
  wazuh-integrations:
  wazuh-active-response:
  wazuh-agentless:
  wazuh-wodles:
  filebeat-etc:
  filebeat-var:
  elastic-data:

networks:
  security-tools-network:
    external: true
```

### Step 2: Deploy the stack

```bash
docker-compose up -d
```

### Step 3: Access the Wazuh dashboard

Open your browser and navigate to `http://your-server-ip:5601`. The default credentials are:
- Username: `elastic`
- Password: `changeme`

## Deploying Vulnerability Scanner: OpenVAS

OpenVAS is a full-featured vulnerability scanner that helps identify security issues in your systems.

### Step 1: Create a docker-compose.yml file

```yaml
version: '3.9'

services:
  openvas:
    image: securecompliance/gvm:latest
    restart: always
    ports:
      - "8080:9392"
    environment:
      - USERNAME=admin
      - PASSWORD=admin
      - RELAYHOST=smtp.example.com
      - SMTPPORT=25
    volumes:
      - openvas-data:/data
    networks:
      - security-tools-network

volumes:
  openvas-data:

networks:
  security-tools-network:
    external: true
```

### Step 2: Deploy OpenVAS

```bash
docker-compose up -d
```

### Step 3: Access the OpenVAS dashboard

The initial setup may take 15-30 minutes. Once complete, access the dashboard at `http://your-server-ip:8080`. Use the credentials specified in the docker-compose file.

## Deploying Web Application Firewall: ModSecurity

ModSecurity is a powerful web application firewall that protects your web applications from various attacks.

### Step 1: Create a docker-compose.yml file

```yaml
version: '3.9'

services:
  modsecurity:
    image: owasp/modsecurity-crs:3.3.2-apache
    restart: always
    ports:
      - "80:80"
      - "443:443"
    environment:
      - PROXY_TIMEOUT=300
      - PARANOIA=1
      - ANOMALY_INBOUND=5
      - ANOMALY_OUTBOUND=4
    volumes:
      - modsec-logs:/var/log/apache2
    networks:
      - security-tools-network

volumes:
  modsec-logs:

networks:
  security-tools-network:
    external: true
```

### Step 2: Deploy ModSecurity

```bash
docker-compose up -d
```

### Step 3: Configure your application to use ModSecurity as a reverse proxy

Update your application's configuration to point to the ModSecurity container as a reverse proxy.

## Troubleshooting

### Common Issues and Solutions

1. **Container fails to start**
   - Check logs: `docker logs <container_name>`
   - Verify resource allocation: `docker stats`

2. **Network connectivity issues**
   - Ensure the security-tools-network exists: `docker network ls`
   - Check container network settings: `docker inspect <container_name> | grep -A 20 "Networks"`

3. **Data persistence issues**
   - Verify volume mounting: `docker inspect <container_name> | grep -A 10 "Mounts"`
   - Check volume permissions: `docker exec <container_name> ls -la /path/to/mounted/volume`

## Security Considerations

1. **Container Hardening**
   - Use official images from trusted sources
   - Regularly update container images
   - Run containers with minimal privileges

2. **Network Security**
   - Limit exposed ports to only what's necessary
   - Use internal Docker networks for inter-container communication
   - Implement network segmentation

3. **Data Protection**
   - Encrypt sensitive data in volumes
   - Implement proper backup procedures for volumes
   - Regularly audit access to data volumes

## Next Steps

- Integrate deployed tools with your existing security infrastructure
- Set up automated updates for container images
- Implement monitoring for container health and performance
- Explore advanced deployment options with Kubernetes for larger environments

By following this tutorial, you've successfully deployed essential security tools using Docker, providing a solid foundation for your security monitoring and protection capabilities.