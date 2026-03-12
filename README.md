Assignment 1 — K3s on AWS
Full Name: Nosipho Nocanda  
Student Number: 223227110  
Repository: assignment-1-Nosipho2022 
Date: March 2026
---
Table of Contents
System Requirements
Architecture Explanation
Environment Setup
Installation Steps
Evidence of Deployment
NGINX Ingress Controller
Storage Configuration
Uninstalling K3s
Troubleshooting
Reflection
---
System Requirements
Component	Specification
Cloud Provider	AWS (us-east-1 / N. Virginia)
Instance Type	t3.large
vCPUs	2 per node
RAM	8 GB per node
Storage	20 GiB gp3 per node
OS	Ubuntu Server 22.04 LTS (64-bit x86)
Nodes	3 × EC2 instances
K3s Version	v1.34.5+k3s1
---
Architecture Explanation
What is K3s?
K3s is a lightweight, CNCF-certified Kubernetes distribution developed by Rancher Labs (now SUSE). It packages the entire Kubernetes control plane into a single binary under 100MB, making it ideal for edge computing, IoT, CI/CD pipelines, and resource-constrained environments. Unlike standard Kubernetes (kubeadm), K3s removes legacy and alpha features, replaces etcd with SQLite by default (or embedded etcd for HA), and bundles everything needed — container runtime, CNI, ingress controller, and storage provisioner — out of the box.
Why K3s on AWS?
Simplicity: Single binary install with one command
HA Support: Embedded etcd enables multi-master HA without external dependencies
Cost-effective: Runs on smaller instance types than full Kubernetes
Production-ready: CNCF certified, suitable for 5G edge and cloud-native workloads
Built-in components: Includes Traefik ingress, Flannel CNI, and local-path storage provisioner
Cluster Architecture
```
┌─────────────────────────────────────────────────────────────┐
│                        AWS VPC (172.31.0.0/16)              │
│                                                             │
│  ┌──────────────────┐  ┌──────────────────┐                │
│  │   k3s-master-1   │  │   k3s-master-2   │                │
│  │  172.31.46.18    │  │  172.31.42.180   │                │
│  │                  │  │                  │                │
│  │  Control Plane   │  │  Control Plane   │                │
│  │  etcd (embedded) │  │  etcd (embedded) │                │
│  │  API Server      │  │  API Server      │                │
│  │  Scheduler       │  │  Scheduler       │                │
│  └────────┬─────────┘  └────────┬─────────┘                │
│           │                     │                           │
│           └──────────┬──────────┘                           │
│                      │ K3s Cluster                          │
│           ┌──────────┴──────────┐                           │
│           │    k3s-master-3     │                           │
│           │   172.31.37.60      │                           │
│           │                     │                           │
│           │   Worker (Agent)    │                           │
│           │   Runs workloads    │                           │
│           └─────────────────────┘                           │
│                                                             │
│  Security Group: k3s-ha-sg                                  │
│  Ports: 22, 6443, 2379-2380, 8472/UDP, 10250, 30000-32767  │
└─────────────────────────────────────────────────────────────┘
```
Key Components
Component	Implementation	Purpose
Control Plane	k3s-master-1, k3s-master-2	API server, scheduler, controller manager
etcd	Embedded (per master)	Cluster state storage — HA with 2 members
Worker/Agent	k3s-master-3	Runs application pods and workloads
Container Runtime	containerd	Runs containers on each node
CNI	Flannel (VXLAN)	Pod networking across nodes (port 8472/UDP)
Ingress	NGINX Ingress Controller	HTTP/HTTPS routing into the cluster
Storage	local-path-provisioner	Dynamic hostPath volume provisioning
Storage (prod)	AWS EBS CSI Driver	Network-attached durable block storage
---
Environment Setup
Step 1 — Create Key Pair (AWS Console)
Navigate to: EC2 → Network & Security → Key Pairs → Create key pair
```
Name:              k3s-key
Key pair type:     RSA
File format:       .pem
```
Set permissions on your local machine:
```bash
chmod 400 ~/Downloads/k3s-key.pem
```
Step 2 — Create Security Group
Navigate to: EC2 → Network & Security → Security Groups → Create security group
```
Name:        k3s-ha-sg
Description: K3s HA cluster security group
VPC:         Default VPC
```
Inbound Rules:
Type	Protocol	Port Range	Source	Purpose
SSH	TCP	22	0.0.0.0/0	Remote access
Custom TCP	TCP	6443	0.0.0.0/0	Kubernetes API server
Custom TCP	TCP	2379-2380	k3s-ha-sg (self)	etcd cluster communication
Custom UDP	UDP	8472	k3s-ha-sg (self)	Flannel VXLAN overlay network
Custom TCP	TCP	10250	k3s-ha-sg (self)	Kubelet API
Custom TCP	TCP	30000-32767	0.0.0.0/0	NodePort services
Step 3 — Launch 3 EC2 Instances
Navigate to: EC2 → Launch instance (repeat 3 times)
```
Names:              k3s-master-1, k3s-master-2, k3s-master-3
AMI:                Ubuntu Server 22.04 LTS (64-bit x86)
Instance type:      t3.large
Key pair:           k3s-key
VPC:                Default
Auto-assign IP:     Enable
Security group:     k3s-ha-sg (select existing)
Storage:            20 GiB gp3
```
Step 4 — Record IP Addresses
Hostname	Private IP	Public IP
k3s-master-1	172.31.46.18	50.17.166.179
k3s-master-2	172.31.42.180	98.83.149.174
k3s-master-3	172.31.37.60	54.80.204.220
> **Note:** AWS Learner Lab assigns dynamic public IPs that change on restart.
---
Installation Steps
Step 5 — Prepare All Nodes
Connect via: EC2 → Instances → Connect → EC2 Instance Connect
Run on each node (change hostname per node):
```bash
# Set hostname (change per node: k3s-master-1 / k3s-master-2 / k3s-master-3)
sudo hostnamectl set-hostname k3s-master-1

# Update packages
sudo apt-get update && sudo apt-get upgrade -y

# Set timezone
sudo timedatectl set-timezone UTC

# Update /etc/hosts on ALL nodes (same content on each)
sudo tee -a /etc/hosts <<EOF
172.31.46.18  k3s-master-1
172.31.42.180  k3s-master-2
172.31.37.60  k3s-master-3
EOF

# Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```
Step 6 — Install K3s Server (master-1 ONLY)
```bash
curl -sfL https://get.k3s.io | sh -s - server \
  --cluster-init \
  --tls-san 50.17.166.179 \
  --tls-san 172.31.46.18
```
Verify K3s is running:
```bash
sudo systemctl status k3s
sudo kubectl get nodes
```
Get the join token:
```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```
Step 6b — Join master-2 as Control Plane
```bash
curl -sfL https://get.k3s.io | sh -s - server \
  --server https://172.31.46.18:6443 \
  --token K10fe4878beebcb29ed1a105a598fc3438d14c7892abb9a289afa5e19463b3cbe96::server:e7980c09f53f2cbb94125efe4b890126 \
  --tls-san 98.83.149.174 \
  --tls-san 172.31.42.180
```
Step 6c — Join master-3 as Agent (Worker)
```bash
curl -sfL https://get.k3s.io | K3S_URL=https://172.31.46.18:6443 \
  K3S_TOKEN=K10fe4878beebcb29ed1a105a598fc3438d14c7892abb9a289afa5e19463b3cbe96::server:e7980c09f53f2cbb94125efe4b890126 \
  sh -
```
Verify all 3 nodes:
```bash
sudo kubectl get nodes -o wide
```
Expected output:
```
NAME           STATUS   ROLES                       AGE   VERSION
k3s-master-1   Ready    control-plane,etcd,master   23m   v1.34.5+k3s1
k3s-master-2   Ready    control-plane,etcd,master   8m    v1.34.5+k3s1
k3s-master-3   Ready    <none>                      4m    v1.34.5+k3s1
```
Deploy test application:
```bash
sudo kubectl create deployment nginx --image=nginx
sudo kubectl expose deployment nginx --port=80 --type=NodePort
sudo kubectl get svc nginx
# Access at http://<PUBLIC_IP>:<NODEPORT>
```
---
Evidence of Deployment
> **Screenshots below show the actual deployment. Terminal prompts display the node hostname confirming originality.**
1. kubectl get nodes — All 3 Nodes Ready
![kubectl get nodes](screenshots/kubectl-get-nodes.png)
2. kubectl get pods -A — All System Pods Running
![kubectl get pods](screenshots/kubectl-get-pods.png)
3. AWS Console — EC2 Instances Running
![EC2 Instances](screenshots/ec2-instances.png)
4. nginx Welcome Page — NodePort Access Confirmed
![nginx](screenshots/nginx-welcome.png)
5. K3s Install Output on master-1
![K3s Install](screenshots/k3s-install.png)
---
NGINX Ingress Controller
K3s deploys Helm charts automatically via manifests placed in `/var/lib/rancher/k3s/server/manifests/`.
```bash
# Create the NGINX Ingress manifest
sudo tee /tmp/nginx-ingress.yaml <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx
---
apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: ingress-nginx
  namespace: kube-system
spec:
  repo: https://kubernetes.github.io/ingress-nginx
  chart: ingress-nginx
  targetNamespace: ingress-nginx
  valuesContent: |-
    controller:
      service:
        type: LoadBalancer
        annotations:
          service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
          service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
EOF

# Deploy it
sudo cp /tmp/nginx-ingress.yaml /var/lib/rancher/k3s/server/manifests/nginx-ingress.yaml

# Verify
sudo kubectl -n ingress-nginx get pods
sudo kubectl -n ingress-nginx get svc
```
---
Storage Configuration
8.1 — Default Storage (local-path-provisioner)
K3s ships with `local-path-provisioner` built-in. Test it:
```bash
# Create PVC
sudo kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF

# Create Pod using the PVC
sudo kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: volume-test
spec:
  containers:
  - name: volume-test
    image: nginx
    volumeMounts:
    - name: test-storage
      mountPath: /data
  volumes:
  - name: test-storage
    persistentVolumeClaim:
      claimName: test-pvc
EOF

# Verify
sudo kubectl get pvc
sudo kubectl get pv
sudo kubectl get pod volume-test
```
8.2 — AWS EBS CSI Driver (Production)
```bash
# Install EBS CSI Driver
sudo kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.35"

# Create EBS StorageClass
sudo kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp3
EOF

# Verify
sudo kubectl get storageclass
sudo kubectl -n kube-system get pods | grep ebs
```
---
Uninstalling K3s
```bash
# On server (master) nodes — master-1 and master-2
sudo /usr/local/bin/k3s-uninstall.sh

# On agent (worker) nodes — master-3
sudo /usr/local/bin/k3s-agent-uninstall.sh

# Verify removal
sudo systemctl status k3s
# Expected: Unit k3s.service could not be found.
```
---
Troubleshooting
Check K3s service logs
```bash
journalctl -u k3s -f
```
Check etcd health
```bash
sudo k3s etcd-snapshot list

# Detailed etcd member list via etcdctl
sudo k3s kubectl -n kube-system exec -it \
  $(sudo k3s kubectl -n kube-system get pod -l component=etcd -o jsonpath='{.items[0].metadata.name}') \
  -- etcdctl --endpoints=https://127.0.0.1:2379 \
     --cacert=/var/lib/rancher/k3s/server/tls/etcd/server-ca.crt \
     --cert=/var/lib/rancher/k3s/server/tls/etcd/client.crt \
     --key=/var/lib/rancher/k3s/server/tls/etcd/client.key \
     member list
```
Nodes stay in NotReady
Check security group rules — ensure ports 8472/UDP (Flannel VXLAN) and 10250/TCP (Kubelet) are open between nodes
Verify hostname resolution: `ping k3s-master-2` from k3s-master-1
API server unreachable
Ensure port 6443/TCP is open in the security group
Verify the `--server` URL points to the correct private IP
Token CA hash mismatch on join
Re-copy the token from master-1: `sudo cat /var/lib/rancher/k3s/server/node-token | tr -d '\n'`
Uninstall K3s on the failing node first: `sudo /usr/local/bin/k3s-uninstall.sh`
Instance Metadata Service (IMDS)
```bash
aws ec2 modify-instance-metadata-options \
  --instance-id <instance-id> \
  --http-put-response-hop-limit 2 \
  --http-endpoint enabled \
  --region us-east-1
```
---
Reflection
What did I learn?
Deploying K3s on AWS taught me the practical difference between theoretical Kubernetes knowledge and real-world implementation. Before this assignment, I understood Kubernetes conceptually — control planes, pods, and services — but had never provisioned a multi-node cluster from scratch. Working through each step, from creating security groups to joining nodes with tokens, gave me hands-on understanding of how these components interact at a network level.
I learned that infrastructure configuration is as important as the software itself. Incorrectly configured security group rules — such as missing the Flannel VXLAN port 8472/UDP — can prevent nodes from communicating entirely. I also learned the importance of using self-referencing security group rules for inter-node traffic, which restricts cluster communication to only the nodes within the group rather than exposing internal ports to the internet.
Working with EC2 Instance Connect instead of SSH keys taught me that cloud-native tools can simplify workflows significantly. Rather than managing key files, I could connect directly from the browser, which is especially useful in learning environments like AWS Learner Labs.
Challenges Faced and How I Resolved Them
The most significant challenge was the token CA hash mismatch when joining master-2 to the cluster. The error message — "token CA hash does not match the Cluster CA certificate hash" — was initially confusing. After checking the logs with `journalctl -xeu k3s.service`, I realized the token had been incorrectly typed (two characters transposed). The fix was to uninstall K3s on master-2, carefully re-copy the exact token from master-1 using `cat | tr -d '\n'` to strip newlines, and reinstall.
Another challenge was attempting to use a `.ppk` key format in AWS CloudShell, which only accepts `.pem` format. This required creating a new key pair and understanding the difference between PuTTY format (.ppk) and OpenSSH format (.pem).
I also encountered a failure when trying to create the security group inline during instance launch — AWS cannot self-reference a security group that does not yet exist. The solution was to create the security group separately first, then select it as an existing group during instance launch.
K3s and Production Kubernetes / 5G Cloud-Native Concepts
K3s is increasingly relevant in 5G and cloud-native telecommunications architectures. In 5G network deployments, Multi-access Edge Computing (MEC) requires Kubernetes clusters to run at the network edge — in base stations, small cells, and edge data centers — where resources are constrained and low latency is critical. K3s's small footprint (single binary, <512MB RAM) makes it ideal for these environments compared to full Kubernetes distributions.
Cloud-native 5G network functions such as AMF (Access and Mobility Management), SMF (Session Management), and UPF (User Plane Function) are containerized and orchestrated by Kubernetes. K3s provides the same Kubernetes API, enabling these network functions to be deployed at the edge with the same tooling used in core data centers. The embedded etcd HA configuration we deployed mirrors the resilience requirements of production 5G core networks, where control plane availability is critical.
Virtualization and Containerization for Scalable Services
This assignment demonstrated how virtualization and containerization work together to enable scalable services. EC2 instances are virtual machines running on AWS's physical hardware — each node is a lightweight OS instance isolated from others. Containerization, via containerd and Kubernetes, adds another layer of abstraction: applications run in containers that share the OS kernel but are isolated in terms of filesystem, network, and process space.
This two-layer approach enables horizontal scaling — adding worker nodes to the cluster scales compute capacity without reconfiguring applications. Kubernetes handles scheduling, load balancing, and self-healing automatically. For 5G and cloud-native services, this means network functions can scale dynamically based on traffic load, ensuring quality of service without manual intervention.
---
Repository: assignment-1-Nosipho2022 | Student: 223227110
