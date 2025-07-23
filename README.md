# Technical Challenge NG-Voice

Juan Pablo Rodriguez Grande
---
## 🚀 Project Overview

This repository provides a step-by-step guide to deploying the classic "Guestbook" application on **Google Kubernetes Engine (GKE)** and **Amazon Elastic Kubernetes Service (EKS)**, complete with a comprehensive monitoring stack using **Prometheus** and **Grafana**. This setup demonstrates a multi-tier web application deployment and its observability in a cloud environment.

**Key Components:**
* **Google Kubernetes Engine (GKE):** Google Cloud's managed Kubernetes service.
* **Amazon Elastic Kubernetes Service (EKS):** Amazon Web Services (AWS)  managed Kubernetes service
* **Guestbook Application:**
    * **Redis Master:** Stores Guestbook entries.
    * **Redis Slaves:** Replicas for read operations.
    * **PHP Frontend:** The web interface for user interaction.
* **Prometheus:** An open-source monitoring system for collecting metrics.
* **Grafana:** An open-source platform for analytics and metric visualization.

---

## 📋 Table of Contents

1.  System Architecture
2.  Prerequisites
3.  Repository Structure
4.  Deployment Guide
    * 4.1. GCP Project Setup
    * 4.2. GKE/EKS Cluster Creation
    * 4.3. Guestbook Application Deployment
    * 4.4. Prometheus & Grafana Deployment
5.  Accessing the Applications
    * 5.1. Accessing the Guestbook
    * 5.2. Accessing Grafana
6.  Cost Optimization (Free Tier / Development)
7.  Resource Cleanup
8.  Important Notes & Considerations
9.  References & Useful Resources
10. Troubleshooting Commands


---

## 1. 🏗️ System Architecture

This diagram illustrates the high-level interaction between the components deployed in your GKE cluster:

```bash
[User]
|
V
[Google/AWS Cloud Load Balancer]
|
V
[GKE/EKS Frontend Pods (PHP)] ---read---> [Redis Slaves Pods]
|                                      ^
|                                      |
----write-----------------------------> [Redis Master Pod]

[Prometheus Pods] <--scrapes metrics from-- [All Cluster Components (Nodes, Pods, Services)]
|
V
[Grafana Pod] <--queries data from-- [Prometheus Pods]
 ```

## 2. 🛠️ Prerequisites

Ensure you have the following installed and configured on your local machine:

* **Google Cloud Platform (GCP) Account:** With billing enabled (First Time you can use free credits). This guide is bases in a new account 
* **Google Cloud SDK (gcloud CLI):**
* **Intallation process**:

    ```bash
    * sudo apt-get update
    * sudo apt-get install apt-transport-https ca-certificates gnupg curl : install http certifiicate transport and curl
    * curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /usr/share/keyrings/cloud.google.gpg : Import the Public Key
    * sudo apt-get update && sudo apt-get install google-cloud-cli : Install cli package
    * gcloud init : start GCLOUD
    ```

    * Authenticate your CLI:
        ```bash
        gcloud auth login
        gcloud config set project YOUR_PROJECT_ID # Replace YOUR_PROJECT_ID
        ```
     * ![image](https://github.com/user-attachments/assets/e4c91e33-dd3b-4132-a42e-0d7f9b804b0d)
     * ![image](https://github.com/user-attachments/assets/46f54a92-f98d-4c71-862c-e8f9d484f8ac)
 
     * Enable google services to interact with GKE
       gcloud services enable container.googleapis.com compute.googleapis.com

       ![image](https://github.com/user-attachments/assets/27fd85be-be4b-491d-a772-7735668f7854)

* **Note:** Ensure your GCP account has the necessary IAM permissions in your project to enable APIs

   ![image](https://github.com/user-attachments/assets/54f2dff5-62f3-466d-a47b-1fc8363e7ed5)

* **Amazon Web Services (AWS) Account**: With billing enabled (Be careful to keep enabled the resorces). This guide is bases in a new account 
* **Set up AWS CLI:**
* **Intallation process**:

    ```bash
    * sudo apt-get update
    * curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.33.0/2025-05-01/bin/linux/amd64/kubectl
    * chmod +x ./kubectl
    * mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$HOME/bin:$PATH
    * echo 'export PATH=$HOME/bin:$PATH' >> ~/.bashrc

    * eksctl installation
    * curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
    * sudo mv /tmp/eksctl /usr/local/bin
    * eksctl version
    ```

    * Authenticate your CLI:
        ```bash
        aws configure
        Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
        Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
        Default region name [None]: region-code
        Default output format [None]: json

        Verification:
        [ec2-user@ip-172-31-81-188 ~]$ aws sts get-caller-identity
        {
            "UserId": "AIDAV5YEEXNL56OWKYYK21",
            "Account": "407493720910",
            "Arn": "arn:aws:iam::407493720910:user/admin"
        }
        ```
      

* **Neccessary Tools**
* **kubectl:** The Kubernetes command-line tool. << Interconnection between cli and GKE to create and admin deployments
    * [kubectl Installation Guide](https://kubernetes.io/docs/tasks/tools/install-kubectl/) (often included with gcloud SDK)
* **Helm (v3+):** The Kubernetes package manager.
    * [Helm Installation Guide](https://helm.sh/docs/intro/install/)
* **Git:** For cloning this repository.(Optional) <<<< This help us to copy the repository in our local machine in case is necessary to edit some files
    * [Git Installation Guide](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)

* **Commands to verify Prerequisites**
```bash
  * gcloud version
  * gcloud projects list
  * aws --version
  * kubectl version
  * helm version
```

* **Expected result:**

![image](https://github.com/user-attachments/assets/575c7f6a-5e9e-407e-b86b-73e83ce81fda)


<img width="984" height="111" alt="image" src="https://github.com/user-attachments/assets/02d2613b-a8d3-404c-aa82-37e23892e83f" />

---

## 3. 📁 Repository Structure

This section outlines the key files and directories within this repository. 

```bash
├── guestbook/
│   ├── frontend-deployment.yaml
│   ├── frontend-service.yaml
│   ├── redis-master-deployment.yaml << Redis help us to store data (DB)
│   ├── redis-master-service.yaml << Redis Service help us to create an external app to communicate with Redis DB to write data
│   ├── redis-slave-deployment.yaml << This help us to have a HA deployment and prevent outage events
│   └── redis-slave-service.yaml << On this file need to focus on point to the master service
├── monitoring/
│   └── prometheus-grafana-values.yaml << This two services help us with monitoring metrics and display information in a web to create and edit dashboards
└── README.md
```

## 4. 🚀 Deployment Guide

Follow these steps to deploy the Guestbook application and the monitoring stack to your GKE cluster.

### 4.1. GCP Project Setup

1.  **Verify if Required APIs are enable:**
    ```bash
    gcloud services list | grep container
    gcloud services list | grep compute
    ```
* **Expected result:**

![image](https://github.com/user-attachments/assets/2233eb10-764a-42eb-8ebd-d4b305d510aa)


### 4.2. GKE/EKS Cluster Creation

To optimize costs as the current account is using free credits, we'll create a small cluster suitable for testing.

1.  **Create the GKE Cluster:**
    ```bash
    gcloud container clusters create guestbook-cluster \
      --project YOUR_PROJECT_ID \ # Replace YOUR_PROJECT_ID
      --region us-central1 \ # Or your preferred region with good quota
      --num-nodes 1 \       # Minimum nodes to save costs
      --machine-type e2-small \ # Economical machine type
      --disk-size 20GB       # Node boot disk size. Adjust if you hit quota issues.
    ```
    **Create the EKS Cluster:**

   ```bash
    eksctl create --name guestbook-cluster \
      --region us-east-1 \
      --nodegroup-name linux-nodes \
      --node-type t3.medium \
      --nodes 2 \
      --nodes-min 1 \
      --nodes-max 3 \
      --managed
   ```
   
* **Expected result:**

    ```bash
    Command used:
    gcloud container clusters create guestbook-cluster --region us-central1 --num-nodes 1 --machine-type e2-small --disk-size 20GB --project august-eye-464222-q4
    ```
![image](https://github.com/user-attachments/assets/28afa165-682e-4592-adb9-d0bf7034166f)

* **Expected result:**

    ```bash
    Command used:
    eksctl create cluster --name guestbook-aws --region us-east-1 --nodegroup-name linux-nodes --node-type t3.medium --nodes 2 --nodes-min 1 --nodes-max 3 --managed
    ```

2.  **Configure `kubectl` to access your cluster:** (This apply for GCP)
    ```bash
    gcloud container clusters get-credentials guestbook-cluster --region us-central1 --project august-eye-464222-q4
    ```
* **Expected result:**

![image](https://github.com/user-attachments/assets/bf20b48a-d663-4220-ac36-0f67858d5a27)

3.  **Verify cluster nodes:**
    ```bash
    kubectl get nodes
    ```
* **Expected result:**

![image](https://github.com/user-attachments/assets/74b4fbf2-18e1-4c4a-aa1e-4c44f8fdf0c5)

### 4.3. Guestbook Application Deployment

First, ensure you have the Guestbook YAML files locally. For this deployment previously we need to copy the repotory files from https://github.com/kubernetes/examples.git or create the files YAML locally and edit using command **vim** or **nano**. In this case we already download the files and edit if its necessary.

1.  **Deploy Redis Master:**
    ```bash
    kubectl apply -f guestbook/redis-master-deployment.yaml
    kubectl apply -f guestbook/redis-master-service.yaml
    ```
* **Expected result:**

```bash
Verify the result pods deployed:
kubectl get pods
kubectl get svc
```
![image](https://github.com/user-attachments/assets/2efd8db9-053a-4b55-b16d-51452f17bded)


2.  **Deploy Redis Slaves:**
    * **Optimization Tip:** For free tier usage, consider editing `guestbook/redis-slave-deployment.yaml` to set `replicas: 1` if it's not already.
    ```bash
    kubectl apply -f guestbook/redis-slave-deployment.yaml
    kubectl apply -f guestbook/redis-slave-service.yaml
    ```
    
* **Expected result:**

```bash
Verify the result pods deployed:
kubectl get pods
kubectl get svc
```
![image](https://github.com/user-attachments/assets/97049a85-df9b-4732-a812-7c611052b25f)


3.  **Deploy Frontend (PHP Guestbook):**
   The Frontend application help us to iniciate the web app using PHP, the interface we will obtain from one of the samples for GKE

     * **Optimization Tip:** Edit guestbook/frontend-deployment.yaml to set replicas: 1.

    ```yaml
    apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: frontend
      spec:
      selector:
      matchLabels:
         app: guestbook
         tier: frontend
         replicas: 1 #<<<<< 
      template:
         metadata:
      labels:
        app: guestbook
        tier: frontend
      spec:
         containers:
         - name: php-redis
         image: gcr.io/google-samples/gb-frontend:v5
      resources:
          requests:
            cpu: 100m
            memory: 100Mi
        env:
        - name: GET_HOSTS_FROM
          value: dns
        ports:
        - containerPort: 80
    ```
     
    * **Crucial:** Ensure guestbook/frontend-service.yaml has **type: LoadBalancer** to expose the application to the internet. 

    ```yaml
    # frontend-service.yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: frontend
      labels:
        app: guestbook
        tier: frontend
    spec:
      type: LoadBalancer # <--- IMPORTANT: Ensure this is LoadBalancer
      ports:
      - port: 80
        targetPort: 80
      selector:
        app: guestbook
        tier: frontend
    ```

    ```bash
    kubectl apply -f guestbook/frontend-deployment.yaml
    kubectl apply -f guestbook/frontend-service.yaml
    ```
* **Expected result:**

```bash
Verify the result pods deployed:
kubectl get pods
kubectl get svc
```
![image](https://github.com/user-attachments/assets/8b6123fa-ad27-47f2-94eb-45f83c768cdd)



### 4.4. Prometheus & Grafana Deployment

1.  **Add the Helm Repository:**
    ```bash
    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    helm repo update
    ```
* **Expected result:**

```bash
Verify the repositories are added correctly:
helm repo list
```

![image](https://github.com/user-attachments/assets/e40e6312-ed8a-46f7-ab8f-41be366c5563)


2.  **Create a Dedicated Namespace:**
    ```bash
    kubectl create namespace monitoring
    ```
* **Expected result:**
  
![image](https://github.com/user-attachments/assets/2557ee8e-12b0-44b0-ac63-c5b9a1ff9dbb)


3.  **Prepare the `monitoring/prometheus-grafana-values.yaml` file:**
    Open the `monitoring/prometheus-grafana-values.yaml` file. This file contains configurations to optimize resource consumption (e.g., disabling Alertmanager, adjusting CPU/Memory requests/limits).
    * **CHANGE THE GRAFANA ADMIN PASSWORD!**
        Find `adminPassword: "yourStrongPassword"` and set a **strong, unique password**.
    * **Grafana Service Type:** By default, it's set to `type: LoadBalancer` for easy external access.

    ```yaml
    # Excerpt from monitoring/prometheus-grafana-values.yaml
    grafana:
      enabled: true
      adminPassword: "admin12#$" # <--- CHANGE THIS!
      service:
        type: LoadBalancer # Or NodePort
    # ... other configurations for Prometheus resource limits, etc.
    ```

4.  **Install `kube-prometheus-stack` with Helm:**
    ```bash
    helm install prometheus prometheus-community/kube-prometheus-stack \
      --namespace monitoring \
      -f monitoring/prometheus-grafana-values.yaml
    ```
* **Expected result:**

```bash
Command used:
helm install prometheus prometheus-community/kube-prometheus-stack --namespace monitoring -f prometheus-grafana-values.yaml
```

![image](https://github.com/user-attachments/assets/88795a7e-e353-46db-91d0-ef08d811ab89)


5.  **Verify monitoring deployment:**
    ```bash
    kubectl get pods -n monitoring
    kubectl get services -n monitoring
    ```
* **Expected result:**

![image](https://github.com/user-attachments/assets/48bf11af-5ae3-4945-9658-9a821902a9d6)

![image](https://github.com/user-attachments/assets/2604f12f-6e6a-41e5-82c8-3b212064ec7a)

---

## 5. 🌐 Accessing the Applications

### 5.1. Accessing the Guestbook

1.  **Get the Frontend's External IP:**
    ```bash
    kubectl get service frontend
    ```
    Wait until the `EXTERNAL-IP` column shows a public IP address (this might take a few minutes).

* **Expected result:**

![image](https://github.com/user-attachments/assets/04be5f3d-bb1a-4514-a311-399c6268c752)

2.  **Open in Browser:**
    Navigate to `http://<GUESTBOOK_EXTERNAL_IP>` in your web browser.

* **Expected result:**

![image](https://github.com/user-attachments/assets/02cde154-73f7-405b-ac15-72540dde2759)


### 5.2. Accessing Grafana

1.  **Get Grafana's External IP:**
    ```bash
    kubectl get service prometheus-grafana -n monitoring
    ```
    Wait until the `EXTERNAL-IP` column shows a public IP address.

* **Expected result:**

![image](https://github.com/user-attachments/assets/a6452cdb-f400-4832-b60a-1c2574030da7)

2.  **Get Grafana Admin Password:**
    The username is `admin`. The password is what you set in `monitoring/prometheus-grafana-values.yaml`. If you need to retrieve it later:
    ```bash
    kubectl get secret --namespace monitoring prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
    ```

3.  **Open in Browser:**
    Navigate to `http://<GRAFANA_EXTERNAL_IP>` in your web browser and log in with the credentials.

* **Expected result:**

![image](https://github.com/user-attachments/assets/5a48a6d2-50b6-4d0b-8e31-8308237e4133)

---


## 6. 💸 Cost Optimization (Free Tier / Development)

This project is configured to minimize costs for a testing environment, but GKE clusters and Load Balancers do incur charges.

* **Small Nodes:** We use `e2-small` and a single node.
* **Minimum Replicas:** Guestbook deployments are configured for a single replica.
* **Prometheus/Grafana Configuration:** The `prometheus-grafana-values.yaml` file disables non-essential components (Alertmanager) and adjusts resource limits.
* **No Data Persistence:** By default, Prometheus data is not persisted to permanent disks, saving storage costs. If you need persistence, you'll need to configure `storageSpec` in `prometheus-grafana-values.yaml`.
* **ALWAYS CLEAN UP!** This is the most crucial step to avoid unexpected charges.

---

## 7. 🧹 Resource Cleanup

To prevent continuous charges on your GCP account, **it is essential to delete all created resources** once you've finished experimenting.

1.  **Uninstall Prometheus & Grafana (Helm):**
    ```bash
    helm uninstall prometheus -n monitoring
    kubectl delete namespace monitoring # Optional, but recommended to delete the namespace
    ```

* **Expected result:**

![image](https://github.com/user-attachments/assets/35107993-1a73-4d3c-bb30-3b823a169d96)


2.  **Delete the Guestbook Application (Kubernetes):**
    ```bash
    kubectl delete -f frontend-service.yaml
    kubectl delete -f frontend-deployment.yaml
    kubectl delete -f redis-slave-service.yaml
    kubectl delete -f redis-slave-deployment.yaml
    kubectl delete -f redis-master-service.yaml
    kubectl delete -f redis-master-deployment.yaml
    ```

* **Expected result:**

![image](https://github.com/user-attachments/assets/588287d9-761a-49c3-9eeb-74ed15c8caed)


3.  **Delete the GKE Cluster:**
    ```bash
    gcloud container clusters delete guestbook-cluster --region us-central1 --project YOUR_PROJECT_ID --quiet
    ```
    * **Verify Billing:** After cleanup, check your GCP Billing dashboard to ensure no residual resources are incurring costs.

* **Expected result:**

    ```bash
    Command used:
    gcloud container clusters delete guestbook-cluster --region us-central1 --project august-eye-464222-q4 --quiet
    ```

![image](https://github.com/user-attachments/assets/e5aa0f42-a0e3-4e44-afb3-25813ad4eca3)

---

## 8. ⚠️ Important Notes & Considerations

* **Grafana Security:** The Grafana admin password is set directly in the `values.yaml` file. For a production environment, never store passwords in plain text in repositories; consider more secure secret management methods (e.g., Google Secret Manager, HashiCorp Vault, encrypted Kubernetes Secrets).
* **Scalability:** This setup is optimized for testing. For a production environment, would be neecessary more nodes, replicas, data persistence, and a more robust resource configuration for Prometheus and Grafana.

---

## 9. 🔗 References & Useful Resources

* [Official Google Kubernetes Engine (GKE) Documentation](https://cloud.google.com/kubernetes-engine/docs)
* [Kubernetes Examples - Guestbook Application](https://github.com/kubernetes/examples/tree/master/guestbook)
* [Official Prometheus Documentation](https://prometheus.io/docs/)
* [Official Grafana Documentation](https://grafana.com/docs/)
* [Prometheus Community Helm Charts - kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
* [gke-gcloud-auth-plugin Installation Guide](https://cloud.google.com/kubernetes-engine/docs/how-to/cluster-access-for-kubectl#install_plugin) (if needed)
* [GCP Quota Monitoring](https://console.cloud.google.com/iam-admin/quotas)
* [GKE Pricing](https://cloud.google.com/kubernetes-engine/pricing)

---

## 10. ⚠️ Troubleshooting Commands

In Kubeclt there are some commands that help us to idetify some failures or debug information about our cluster and deployments also help us to identify or describe additional information about our app/deployment/cluster here are some commands:

- Check memory and CPU usage -- kubectl top pod
- Decribe pod -- kubectl describe pod <NAME>
- kubectl logs <pod-name>
- kubectl get nodes -o wide
- kubectl debug <pod-name>
- kubectl get node <my node> -o yaml
- kubectl debug node/mynode -it --image=ubuntu (Valid in ubuntu)
---
