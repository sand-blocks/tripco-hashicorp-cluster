# HashiCorp Cluster Infrastructure

## Overview

This project provides a complete, production-ready infrastructure solution built on the HashiCorp stack. It deploys a 3-node cluster with Nomad and Consul for container orchestration and service discovery, all automated using Ansible.

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Load Balancer (Nginx)                    │
│                  HTTP Traffic Distribution                  │
└─────────────────────────────────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
┌───────▼────────┐  ┌──────▼──────┐  ┌────────▼────────┐
│   Node 1       │  │   Node 2    │  │    Node 3       │
├────────────────┤  ├─────────────┤  ├─────────────────┤
│ • Consul       │  │ • Consul    │  │ • Consul        │
│ • Nomad        │  │ • Nomad     │  │ • Nomad         │
│ • Docker       │  │ • Docker    │  │ • Docker        │
│ • Node.js App  │  │ • Node.js   │  │ • Node.js App   │
└────────────────┘  └─────────────┘  └─────────────────┘
```

### Technology Stack

- **HashiCorp Nomad**: Container orchestration and workload scheduling
- **HashiCorp Consul**: Service discovery, health checking, and distributed key-value store
- **Docker**: Application containerization
- **Nginx**: High-performance load balancer with caching capabilities
- **Ansible**: Infrastructure automation and configuration management
- **Node.js**: Sample web application backend

## Key Features

### 1. **Automated Provisioning**
- Complete infrastructure deployment from bare-metal servers
- Idempotent Ansible playbooks for consistent environments
- Zero-touch cluster setup and configuration

### 2. **High Availability**
- 3-node cluster for redundancy
- Automatic failover and service recovery
- Distributed consensus via Consul's Raft protocol

### 3. **Service Discovery**
- Automatic service registration with Consul
- DNS-based service discovery
- Health checking and automatic de-registration of unhealthy services

### 4. **Load Balancing**
- Nginx reverse proxy for traffic distribution
- HTML response caching (configurable TTL)
- Health-aware routing

### 5. **Container Orchestration**
- Declarative job specifications
- Rolling updates with zero downtime
- Resource constraints and scheduling
- Auto-restart on failures

### 6. **Security**
- Firewall configuration (UFW)
- Fail2ban for brute-force protection
- Non-root container execution
- ACL-ready infrastructure

## Project Structure

```
.
├── README.md
├── config/ansible/
│                                        # Ansible configuration
├── webapp/
│   ├── Dockerfile                       # Node.js app container
│   └── index.js                         # Web application code
```

## Prerequisites

### Control Machine (Where you run Ansible)
- Ansible 2.10 or higher
- Python 3.12+
- SSH access to target nodes

### Target Nodes (3 servers)
- Ubuntu 22.04 or 24.04 LTS
- Root or sudo access
- Network connectivity between all nodes

### Network Requirements
- Open ports between nodes:
  - **Consul**: 8300, 8301, 8302, 8500, 8600
  - **Nomad**: 4646, 4647, 4648
  - **SSH**: 22
  - **HTTP/HTTPS**: 80, 443

---

## Quick Start Tutorial

This tutorial will guide you through deploying the complete HashiCorp cluster from scratch.

### Step 1: Prepare Your Environment

**1.1 Install Ansible on your control machine**

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install -y ansible python3-pip

# Verify installation
ansible --version
```

**1.2 Clone the repository**

```bash
git clone https://github.com/sand-blocks/tripco-hashicorp-cluster.git
cd tripco-hashicorp-cluster
```

**1.3 Configure Hosts file**

Ensure that the neccessary server information has been configured in the `config/ansible/hosts.ini` file:

```bash
cd config/ansible/
cp hosts.ini_SAMPLE hosts.ini
```

Edit the hosts file with the valid credentials

```
[servers]
nomad-server-1 nomad_type="server" ansible_ssh_host="SSH_IP" ansible_ssh_user="SSH_USER" ansible_ssh_pass="SSH_PASSWORD" subnet_ip="PRIVATE_SUBNET_IP"
nomad-server-2 nomad_type="server" ansible_ssh_host="SSH_IP" ansible_ssh_user="SSH_USER" ansible_ssh_pass="SSH_PASSWORD" subnet_ip="PRIVATE_SUBNET_IP"
nomad-server-3 nomad_type="server" ansible_ssh_host="SSH_IP" ansible_ssh_user="SSH_USER" ansible_ssh_pass="SSH_PASSWORD" subnet_ip="PRIVATE_SUBNET_IP"
```


**2 Test connectivity**

```bash
cd config/ansible
ansible -i hosts.ini all -m ping
```

Expected output:
```
nomad-server-1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
nomad-server-2 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
nomad-server-3 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

### Step 3: Provision the Cluster

**3.1 Run the main provisioning playbook**

This installs Docker, Consul, Nomad, fail2ban and UFW on all nodes:

```bash
cd config/ansible
ansible-playbook -i hosts.ini provision_cluster.yaml
```

This will take 5-10 minutes. The playbook will:
- ✓ Update system packages
- ✓ Install Docker on all nodes
- ✓ Install and configure Consul agents
- ✓ Install and configure Nomad servers/clients
- ✓ Set up firewall rules
- ✓ Configure security hardening

**3.2 Verify the cluster (OPTIONAL)**

The outcome of the previous step should provide all neccessary information as to the outcome of the cluster provisioning
```bash
# SSH to any node
ssh ubuntu@nomad-server-1

# Check Consul cluster
consul members
# Expected output: Shows all 3 nodes as "alive"

# Check Nomad servers
nomad server members
# Expected output: Shows 3 servers with one as "leader"

# Check Nomad clients
nomad node status
# Expected output: Shows all nodes as "ready"

# Check Docker
docker ps
# Expected output: No errors
```

### Step 4: Deploy the Node.js Application

**4.1 Build the Docker image**

The docker image has been pre-built and is hosted on Docker Hub for ease of use

[https://hub.docker.com/r/charlin/tripco-webapp](https://hub.docker.com/r/charlin/tripco-webapp)

**4.3 Deploy with Ansible**

```bash
cd config/ansible
ansible-playbook -i hosts.ini deploy_webapp.yaml
```

**4.4 Verify deployment**

Navigate to the public URL of any of the cluster nodes on PORT 8080 [http://nomad-server-1:8080](#)

or use your terminal

```bash
curl http://node1-ip:8080
curl http://node2-ip:8080
curl http://node3-ip:8080
```

### Step 5: Deploy the Load Balancer

**5.1 Deploy Nginx load balancer**

```bash
cd config/ansible
ansible-playbook -i hosts.yaml deploy_loadbalancer.yaml
```

This configures Nginx on node1 (or dedicated load balancer) with:
- Upstream pool pointing to all application instances
- HTML response caching (60-second TTL)
- Health checks

**5.2 Test load balancing**
Navigate to the public URL of any of the cluster nodes on PORT 80 [http://nnomad-server-1:80](#)

You should notice that the result is cached and the time does not change, the container will change every so often


### Step 6: Bootstrap ACLs (Optional)

**6.1 Bootstrap Consul ACLs**

```bash
cd config/ansible
ansible-playbook -i hosts.ini consul_acl_bootstrap.yml
```

This will:
- Initialize Consul ACL system
- Create bootstrap token
- Set up default policies
- Save tokens to hashicorp vault*

**6.2 Bootstrap Nomad ACLs**

```bash
ansible-playbook -i hosts.ini nomad_acl_bootstrap.yml 
```

This will:
- Initialize Nomad ACL system
- Create bootstrap token
- Set up default policies
- Save tokens to hashicorp vault* 

**⚠️ IMPORTANT**: Store these tokens securely! They provide full administrative access.

### Step 7: Verify Complete Setup

**7.1 Access the UIs**

Open in your browser:
- **Consul UI**: http://nomad-server-1:8500
- **Nomad UI**: http://nomad-server-1:4646
- **Application**: http://nomad-server-1/

**7.2 Run health checks**

```bash
# Check all services
ssh ubuntu@nomad-server-1

consul catalog services
nomad status
docker ps

# Check system resources
nomad node status -verbose
```

**7.3 Test high availability**

```bash
# Drain one node
nomad node drain -enable <node-id>

# Application should still be accessible
curl http://nomad-server-X/

# Check job reallocation
nomad job status webapp-backend

# Re-enable node
nomad node drain -disable <node-id>
```

---

## Common Operations

### Scaling Applications

```bash
# Scale up the webapp
ssh ubuntu@nomad-server-1
nomad job scale webapp-backend 6

# Scale down
nomad job scale webapp-backend 2
```

---

## Troubleshooting

### Consul Issues

**Problem**: Consul cluster won't form

```bash
# Check Consul logs
journalctl -u consul -n 50

# Check network connectivity
consul members -detailed

# Verify ports are open
sudo ufw status
sudo netstat -tlnp | grep consul
```

**Solution**: Ensure firewall rules allow Consul ports (8300-8302, 8500, 8600)

### Nomad Issues

**Problem**: Nomad server not joining cluster

```bash
# Check Nomad logs
journalctl -u nomad -n 50

# Verify Consul is running
systemctl status consul

# Check Nomad configuration
nomad agent-info
```

**Solution**: Nomad requires Consul to be running and healthy first

### Docker Issues

**Problem**: Containers not starting

```bash
# Check Docker status
systemctl status docker

# View container logs
docker logs <container-id>

# Check Docker storage
df -h /var/lib/docker
```

### Application Issues

**Problem**: Load balancer returns 502 Bad Gateway

```bash
# Check backend health in Consul
consul catalog service webapp-backend

# Check Nomad allocations
nomad job allocs webapp-backend

# Test backend directly
curl http://nomad-server-1:8080
```

**Solution**: Ensure webapp allocations are running and healthy

---

## Security Considerations

### Current Security Features

- ✓ UFW firewall configured
- ✓ Fail2ban for SSH protection
- ✓ Non-root Docker containers
- ✓ Minimal attack surface

### Recommended Enhancements

1. **Enable TLS**
   - Consul gossip encryption
   - Nomad TLS certificates
   - Nginx SSL termination

2. **Enable ACLs**
   - Run bootstrap playbook
   - Implement token-based access
   - Set up policies per team

3. **Network Segmentation**
   - Use private network only for cluster communication
   - Expose only load balancer to public

4. **Secrets Management**
   - Integrate HashiCorp Vault
   - Rotate credentials regularly

---

## Orchastration Optimization

1. **Implement CI/CD DevOps Best Practices

There should be a well defined workflow which ensures the version control repositories are monitored and automatically triggered when changes need to take places. While this is somewhat of an automated approach, proper seperation of concerns for IaC, Configuration Management, CI/CD pipelines could definitely be leveraged.


---

## Next Steps

After completing this tutorial, consider:

1. **Implement monitoring** with Prometheus and Grafana
2. **Implement logging** with ELK stack
2. **Add Vault** for secrets management
3. **Enable ACLs** for production security
4. **Set up CI/CD** pipeline for automated deployments
5. **Configure backups** for disaster recovery
6. **Add more applications** to the cluster