# K3S Deployment Guide: Kubernetes Cluster on AWS EC2

This guide provides step-by-step instructions to deploy a Kubernetes cluster using K3S with 1 master node and 2 worker nodes on AWS EC2 Ubuntu instances.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Architecture Overview](#architecture-overview)
- [Step 1: Set Up AWS EC2 Instances](#step-1-set-up-aws-ec2-instances)
- [Step 2: Configure Security Groups](#step-2-configure-security-groups)
- [Step 3: Install K3S on Master Node](#step-3-install-k3s-on-master-node)
- [Step 4: Install K3S on Worker Nodes](#step-4-install-k3s-on-worker-nodes)
- [Step 5: Verify Cluster Status](#step-5-verify-cluster-status)
- [Step 6: Deploy Sample Application](#step-6-deploy-sample-application)
- [Troubleshooting](#troubleshooting)
- [Cleanup](#cleanup)

## Prerequisites

Before starting, ensure you have:

- **AWS Account**: Active AWS account with permissions to create EC2 instances
- **SSH Key Pair**: AWS key pair for SSH access to EC2 instances
- **Basic Knowledge**: Familiarity with Linux command line and Kubernetes concepts
- **Three Ubuntu EC2 Instances**:
  - `node1` (master): t2.medium or larger (2 vCPU, 4 GB RAM minimum)
  - `node2` (worker): t2.small or larger (1 vCPU, 2 GB RAM minimum)
  - `node3` (worker): t2.small or larger (1 vCPU, 2 GB RAM minimum)
- **Operating System**: Ubuntu 20.04 LTS or Ubuntu 22.04 LTS

## Architecture Overview

```
┌─────────────────────────────────────────────────┐
│              K3S Cluster                        │
│                                                 │
│  ┌──────────────┐                              │
│  │    node1     │  (Master Node)               │
│  │  K3S Server  │  - API Server                │
│  │              │  - Scheduler                 │
│  │              │  - Controller Manager        │
│  └──────────────┘                              │
│         │                                       │
│         ├─────────────┬─────────────┐          │
│         │             │             │          │
│  ┌──────────────┐ ┌──────────────┐            │
│  │    node2     │ │    node3     │            │
│  │  K3S Agent   │ │  K3S Agent   │            │
│  │ (Worker)     │ │ (Worker)     │            │
│  └──────────────┘ └──────────────┘            │
│                                                 │
└─────────────────────────────────────────────────┘
```

## Step 1: Set Up AWS EC2 Instances

### 1.1 Launch EC2 Instances

1. **Log in to AWS Console** and navigate to EC2 Dashboard
2. **Launch Instances** with the following configuration:

   **For Master Node (node1):**
   - AMI: Ubuntu Server 22.04 LTS
   - Instance Type: t2.medium (minimum)
   - Storage: 20 GB gp3
   - Key Pair: Select your existing key pair
   - Network: Default VPC
   - Tags:
     - Name: k3s-master-node1
     - Role: master

   **For Worker Nodes (node2 and node3):**
   - AMI: Ubuntu Server 22.04 LTS
   - Instance Type: t2.small (minimum)
   - Storage: 20 GB gp3
   - Key Pair: Same as master
   - Network: Same VPC as master
   - Tags:
     - Name: k3s-worker-node2 / k3s-worker-node3
     - Role: worker

### 1.2 Note Private IP Addresses

After instances are running, note down the private IP addresses:
```bash
# Example:
node1 (master): 172.31.10.100
node2 (worker): 172.31.10.101
node3 (worker): 172.31.10.102
```

## Step 2: Configure Security Groups

### 2.1 Create Security Group Rules

Configure the security group attached to your instances with the following rules:

**Inbound Rules:**

| Type | Protocol | Port Range | Source | Description |
|------|----------|------------|--------|-------------|
| SSH | TCP | 22 | Your IP | SSH access |
| Custom TCP | TCP | 6443 | Security Group | Kubernetes API |
| Custom TCP | TCP | 10250 | Security Group | Kubelet metrics |
| Custom TCP | TCP | 8472 | Security Group | Flannel VXLAN |
| Custom UDP | UDP | 8472 | Security Group | Flannel VXLAN |
| Custom TCP | TCP | 2379-2380 | Security Group | etcd |
| Custom TCP | TCP | 30000-32767 | 0.0.0.0/0 | NodePort Services (optional) |

**Outbound Rules:**
- Allow all outbound traffic

### 2.2 Update /etc/hosts (Optional but Recommended)

On each node, update `/etc/hosts` for easier hostname resolution:

```bash
# SSH to each node and run:
sudo nano /etc/hosts
```

Add the following entries (replace with your actual private IPs):
```
172.31.10.100 node1 k3s-master
172.31.10.101 node2 k3s-worker1
172.31.10.102 node3 k3s-worker2
```

## Step 3: Install K3S on Master Node

### 3.1 SSH to Master Node

```bash
ssh -i your-key.pem ubuntu@<master-public-ip>
```

### 3.2 Update System

```bash
sudo apt update && sudo apt upgrade -y
```

### 3.3 Install K3S Server

```bash
curl -sfL https://get.k3s.io | sh -s - server \
  --node-name node1 \
  --write-kubeconfig-mode 644
```

**Installation Options Explained:**
- `server`: Installs K3S in server (master) mode
- `--node-name node1`: Sets the node name
- `--write-kubeconfig-mode 644`: Makes kubeconfig readable by all users

### 3.4 Verify K3S Server Installation

```bash
# Check K3S service status
sudo systemctl status k3s

# Check node status
sudo kubectl get nodes

# Expected output:
# NAME    STATUS   ROLES                  AGE   VERSION
# node1   Ready    control-plane,master   30s   v1.28.x+k3s1
```

### 3.5 Retrieve Node Token

The node token is required for worker nodes to join the cluster:

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

**Save this token** - you'll need it for worker nodes. Example token:
```
K10abc123def456ghi789jkl012mno345pqr678stu901vwx234yz::server:abc123def456ghi789
```

### 3.6 Get Master Node IP

```bash
# Get private IP
hostname -I | awk '{print $1}'
```

## Step 4: Install K3S on Worker Nodes

Repeat these steps for both `node2` and `node3`.

### 4.1 SSH to Worker Node

```bash
ssh -i your-key.pem ubuntu@<worker-public-ip>
```

### 4.2 Update System

```bash
sudo apt update && sudo apt upgrade -y
```

### 4.3 Install K3S Agent

Replace `<MASTER_IP>` with your master node's private IP and `<NODE_TOKEN>` with the token from step 3.5:

**For node2:**
```bash
curl -sfL https://get.k3s.io | K3S_URL=https://<MASTER_IP>:6443 \
  K3S_TOKEN=<NODE_TOKEN> \
  sh -s - agent --node-name node2
```

**For node3:**
```bash
curl -sfL https://get.k3s.io | K3S_URL=https://<MASTER_IP>:6443 \
  K3S_TOKEN=<NODE_TOKEN> \
  sh -s - agent --node-name node3
```

**Example:**
```bash
curl -sfL https://get.k3s.io | K3S_URL=https://172.31.10.100:6443 \
  K3S_TOKEN=K10abc123def456ghi789jkl012mno345pqr678stu901vwx234yz::server:abc123def456ghi789 \
  sh -s - agent --node-name node2
```

### 4.4 Verify Agent Installation

```bash
# Check K3S agent service status
sudo systemctl status k3s-agent
```

## Step 5: Verify Cluster Status

### 5.1 Check All Nodes (from Master Node)

SSH back to the master node and verify all nodes have joined:

```bash
sudo kubectl get nodes -o wide
```

**Expected output:**
```
NAME    STATUS   ROLES                  AGE     VERSION        INTERNAL-IP     EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
node1   Ready    control-plane,master   5m30s   v1.28.x+k3s1   172.31.10.100   <none>        Ubuntu 22.04.x LTS   5.15.0-xx-generic   containerd://1.7.x
node2   Ready    <none>                 2m15s   v1.28.x+k3s1   172.31.10.101   <none>        Ubuntu 22.04.x LTS   5.15.0-xx-generic   containerd://1.7.x
node3   Ready    <none>                 1m45s   v1.28.x+k3s1   172.31.10.102   <none>        Ubuntu 22.04.x LTS   5.15.0-xx-generic   containerd://1.7.x
```

### 5.2 Check System Pods

```bash
sudo kubectl get pods -A
```

**Expected output:**
```
NAMESPACE     NAME                                     READY   STATUS    RESTARTS   AGE
kube-system   coredns-xxxxx                           1/1     Running   0          5m
kube-system   local-path-provisioner-xxxxx            1/1     Running   0          5m
kube-system   metrics-server-xxxxx                    1/1     Running   0          5m
kube-system   svclb-traefik-xxxxx                     2/2     Running   0          4m
kube-system   traefik-xxxxx                           1/1     Running   0          4m
```

### 5.3 Check Cluster Info

```bash
sudo kubectl cluster-info

# Expected output:
# Kubernetes control plane is running at https://127.0.0.1:6443
# CoreDNS is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
# Metrics-server is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/https:metrics-server:https/proxy
```

## Step 6: Deploy Sample Application

Let's deploy a simple nginx application to verify the cluster is working correctly.

### 6.1 Create Deployment

```bash
sudo kubectl create deployment nginx --image=nginx:latest --replicas=3
```

### 6.2 Expose Deployment

```bash
sudo kubectl expose deployment nginx --type=NodePort --port=80
```

### 6.3 Verify Deployment

```bash
# Check deployment
sudo kubectl get deployments

# Check pods
sudo kubectl get pods -o wide

# Check service
sudo kubectl get services
```

**Expected output:**
```
NAME    TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
nginx   NodePort   10.43.xxx.xxx   <none>        80:30xxx/TCP   30s
```

### 6.4 Test Application

```bash
# Get the NodePort
NODE_PORT=$(sudo kubectl get svc nginx -o jsonpath='{.spec.ports[0].nodePort}')
echo "NodePort: $NODE_PORT"

# Test from master node
curl http://localhost:$NODE_PORT

# Expected output: Nginx welcome page HTML
```

You can also access the application from your browser using any node's public IP:
```
http://<node-public-ip>:<node-port>
```

### 6.5 Clean Up Sample Application

```bash
sudo kubectl delete service nginx
sudo kubectl delete deployment nginx
```

## Troubleshooting

### Issue 1: Worker Node Not Joining Cluster

**Symptoms:** Worker node status shows "NotReady" or doesn't appear in node list

**Solutions:**
```bash
# On worker node, check K3S agent logs
sudo journalctl -u k3s-agent -f

# Verify master node is reachable
ping <MASTER_IP>
telnet <MASTER_IP> 6443

# Verify correct token is used
sudo cat /etc/systemd/system/k3s-agent.service.env

# Restart K3S agent
sudo systemctl restart k3s-agent
```

### Issue 2: Security Group Configuration

**Symptoms:** Connection timeout when worker tries to join

**Solution:**
- Verify security group allows traffic on port 6443 from worker nodes
- Check VPC routing tables
- Ensure all nodes are in the same VPC or have proper peering

### Issue 3: Pods Stuck in Pending State

**Symptoms:** Pods remain in "Pending" status

**Solutions:**
```bash
# Check pod events
sudo kubectl describe pod <pod-name>

# Check node resources
sudo kubectl top nodes

# Check for taints
sudo kubectl describe nodes | grep -i taint
```

### Issue 4: DNS Resolution Issues

**Symptoms:** Pods cannot resolve DNS names

**Solutions:**
```bash
# Check CoreDNS pods
sudo kubectl get pods -n kube-system | grep coredns

# Test DNS from a pod
sudo kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup kubernetes.default

# Restart CoreDNS
sudo kubectl rollout restart deployment coredns -n kube-system
```

### Issue 5: Insufficient Resources

**Symptoms:** Nodes showing "NotReady" or pods evicted

**Solutions:**
```bash
# Check node resources
sudo kubectl describe node <node-name>

# Check disk space on all nodes
df -h

# Check memory
free -h

# Consider upgrading instance types if resources are insufficient
```

## Useful Commands

### Cluster Management
```bash
# View cluster info
sudo kubectl cluster-info

# Get all resources
sudo kubectl get all -A

# Check K3S version
k3s --version

# View kubeconfig
sudo cat /etc/rancher/k3s/k3s.yaml
```

### Node Management
```bash
# Drain a node (before maintenance)
sudo kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Uncordon a node (after maintenance)
sudo kubectl uncordon <node-name>

# Remove a node from cluster
sudo kubectl delete node <node-name>
```

### Log Viewing
```bash
# Master node logs
sudo journalctl -u k3s -f

# Worker node logs
sudo journalctl -u k3s-agent -f

# Pod logs
sudo kubectl logs <pod-name> -n <namespace>
```

### Configuration
```bash
# View K3S configuration
sudo cat /etc/systemd/system/k3s.service.env
sudo cat /etc/systemd/system/k3s-agent.service.env

# Edit K3S service (master)
sudo systemctl edit k3s

# Edit K3S service (worker)
sudo systemctl edit k3s-agent
```

## Cleanup

To completely remove the K3S cluster:

### On Worker Nodes (node2 and node3)

```bash
# Stop and uninstall K3S agent
sudo /usr/local/bin/k3s-agent-uninstall.sh
```

### On Master Node (node1)

```bash
# Stop and uninstall K3S server
sudo /usr/local/bin/k3s-uninstall.sh
```

### On AWS Console

1. Terminate all EC2 instances
2. Delete associated security groups (if not using default)
3. Remove any created key pairs (if no longer needed)

## Additional Resources

- [K3S Official Documentation](https://docs.k3s.io/)
- [K3S GitHub Repository](https://github.com/k3s-io/k3s)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [AWS EC2 Documentation](https://docs.aws.amazon.com/ec2/)

## Best Practices

1. **Security:**
   - Use private subnets for worker nodes when possible
   - Restrict SSH access to specific IP ranges
   - Use IAM roles for EC2 instances instead of access keys
   - Enable AWS CloudWatch for monitoring
   - Regularly update K3S and Ubuntu packages

2. **High Availability:**
   - For production, use multiple master nodes (HA setup)
   - Use external database (MySQL, PostgreSQL) instead of embedded etcd
   - Implement proper backup strategy for etcd data

3. **Monitoring:**
   - Install Prometheus and Grafana for monitoring
   - Set up log aggregation (ELK stack or CloudWatch)
   - Monitor node resources and set up alerts

4. **Networking:**
   - Use AWS Load Balancers for exposing services
   - Consider using AWS VPC CNI for better network performance
   - Plan IP address ranges to avoid conflicts

5. **Storage:**
   - Use AWS EBS for persistent storage
   - Consider using AWS EFS for shared storage
   - Implement regular backup strategies

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Contributing

Contributions are welcome! Please feel free to submit issues or pull requests.

## Support

For issues or questions:
- Open an issue in this repository
- Check the [K3S Slack channel](https://slack.rancher.io/)
- Review K3S documentation and GitHub issues