# 🗳️ K8s Kind Voting App (GitOps with ArgoCD)

![Kubernetes](https://img.shields.io/badge/kubernetes-%23326ce5.svg?style=for-the-badge&logo=kubernetes&logoColor=white)
![ArgoCD](https://img.shields.io/badge/ArgoCD-%23EF7B4D.svg?style=for-the-badge&logo=argo&logoColor=white)
![Docker](https://img.shields.io/badge/docker-%230db7ed.svg?style=for-the-badge&logo=docker&logoColor=white)
![AWS](https://img.shields.io/badge/AWS-%23FF9900.svg?style=for-the-badge&logo=amazon-aws&logoColor=white)

A comprehensive guide for setting up a Kubernetes cluster using **Kind** on an **AWS EC2** instance, installing and configuring **Argo CD**, and deploying a microservices-based Voting Application using GitOps principles.

---

## 📖 Overview

This project demonstrates an end-to-end DevOps workflow. The guide covers the steps to:
- Launch an AWS EC2 instance.
- Install Docker and Kind.
- Install and access kubectl.
- Create a Kubernetes cluster using Kind.
- Install and configure Argo CD.
- Connect and manage your Kubernetes cluster with Argo CD.
- Set up the Kubernetes Dashboard.

### Application Components
The application uses a microservices architecture:
* **Vote App:** A front-end web app in [Python](/vote) which lets you vote between two options.
* **Redis:** A [Redis](https://hub.docker.com/_/redis/) instance which collects new votes.
* **Worker:** A [.NET](/worker/) worker which consumes votes and stores them in the database.
* **Postgres:** A [Postgres](https://hub.docker.com/_/postgres/) database backed by a Docker volume.
* **Result App:** A [Node.js](/result) web app which shows the results of the voting in real time.

> **📂 Note:** All screenshots and architecture diagrams referenced in this documentation are stored in the [`Screenshot`](./Screenshots/) folder.
---
---

## 🏗️ Architecture

![Architecture diagram](./Screenshots/k8s-kind-voting-app.png)

---

## 📊 Observability

Monitoring the cluster and application performance is handled via Prometheus and Grafana.

![Grafana diagram](./Screenshots/grafana.png)
---
![Prometheus diagram](./Screenshots/prometheus.png)

---

## 🛠️ Prerequisites & Infrastructure Setup

### 1. AWS EC2 Instance
* **OS:** Ubuntu 22.04 LTS
* **Type:** `t2.medium` (Recommended: 2 vCPU, 4GB RAM)

### 2. Installation Script
Run the following commands on your EC2 instance to install Docker, Kind, and Kubectl:

```bash
# Install Docker
sudo apt-get update
sudo apt-get install docker.io -y
sudo usermod -aG docker $USER && newgrp docker

# Install Kind
cd kind-cluster
chmod +x install_kind.sh
./install_kind.sh

# Install Kubectl
cd kind-cluster
chmod +x install_kubectl.sh
./install_kubectl.sh
````

-----

## 🚀 Getting Started

### 1\. Clone the Repository

```bash
git clone https://github.com/Vaibhav0643/project_k8s-voting-app.git
cd /project_k8s-voting-app
```

### 2\. Set up the Kubernetes Cluster

We will use `kind` to spin up a cluster.

```bash
# Create a cluster
cd kind-cluster
kind create cluster --config=config.yml --name voting-app-cluster

# Verify the cluster
kubectl get nodes
docker ps
```

### 3\. Install ArgoCD

Deploy ArgoCD in the `argocd` namespace.

```bash
# Create namespace
kubectl create namespace argocd

# Apply ArgoCD Manifests
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Check services in Argo CD namespace:
kubectl get svc -n argocd
```

### 4\. Access ArgoCD UI

By default, the ArgoCD server is not exposed externally. We can patch it to use a **NodePort** and then use **Port Forwarding**.

**Step 1: Patch Service OR Expose Argo CD server using NodePort**

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
```
**Step 2: Port Forwarding**

```bash
kubectl port-forward service/argocd-server -n argocd 8443:443 --address=0.0.0.0 &
```

*You can now access ArgoCD at `http://<EC2 Public IP>:<NodePort>`.*  
> **Note:** You have to expose port 8443 in inbound rules of the security group.

### 5\. Login to ArgoCD

  * **Username:** `admin`
  * **Password:** Retrieve the initial password using:
    ```bash
    kubectl get secret -n argocd  argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
    ```

-----

## 🔄 Deploying the Application (GitOps Flow)

1.  Open the **ArgoCD UI**.
2.  Click **+ New App**.
3.  Configure the application:
      * **Application Name:** `voting-app`
      * **Project:** `default`
      * **Sync Policy:** `Automatic` (Enables auto sync/auto-deployment)
      * Enable `PRUNE RESOURCES` : Delete old resources
      * Enable `SELF HEAL` : Create Pods if deleted manually or crashed
      * **Repository URL:** `https://github.com/Vaibhav0643/project_k8s-voting-app.git`
      * **Revision** : `main`
      * **Path:** `k8s-specifications` (or root `./` if manifests are in root)
      * **Cluster URL:** `https://kubernetes.default.svc`
      * **Namespace:** `default`
4.  Click **Create**.

ArgoCD will now sync the state from the GitHub repository to your cluster. You should see the application health turn **Healthy** and **Synced**.

-----

## 🧪 Testing the Deployment

Once synced, check the running services:

```bash
kubectl get svc
```
Test the Auto-sync:  
* Increase the no. of replica for result app in 
[result-deployment.yaml](/k8s-specifications/result-deployment.yaml/)
* Commit the changes from GitHub.
* Check the Replica of result app in ArgoCD UI. It will show 3 replicas of result.

> **📂 Note:** All screenshots and architecture diagrams referenced in this documentation are stored in the [`Screenshot`](./Screenshots/) folder.


You can access the application via port forwarding:

* **Vote App (Python):**

  ```bash
  kubectl port-forward svc/vote 5000:5000 &
  # Open http://<EC2 Public IP>:5000
  ```

* **Result App (Node.js):**

  ```bash
  kubectl port-forward svc/result 5001:5001 &
  # Open http://<EC2 Public IP>:5001
  ```
> **Note:** You have to expose port 5000 & 5001 in inbound rules of the security group.

## Installing Kubernetes Dashboard

- Deploy Kubernetes dashboard:
  ```bash
  kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml

  cd kind-cluster
  kubctl apply -f kind-cluster\dashboard-adminuser.yml
  ```

- Create a token for dashboard access:
  ```bash
  kubectl -n kubernetes-dashboard create token admin-user
  # Copy this token
  ```
- Access the Kubernetes Dashboard using port forwarding:
  ```bash
  kubectl port-forward svc/vote -n kubernetes-dashboard 8080:443 &
  # Open http://<EC2 Public IP>:8080
  # Paste token
  ```
> **Note:** You have to expose port 8080 in inbound rules of the security group.


---


## 🤝 Contributing

1.  Fork the Project
2.  Create your Feature Branch (`git checkout -b feature/AmazingFeature`)
3.  Commit your Changes (`git commit -m 'Add some AmazingFeature'`)
4.  Push to the Branch (`git push origin feature/AmazingFeature`)
5.  Open a Pull Request
