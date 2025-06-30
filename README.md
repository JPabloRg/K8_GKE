# Technical Challenge NG-Voice

---
## 🚀 Project Overview

This repository provides a step-by-step guide to deploying the classic "Guestbook" application on **Google Kubernetes Engine (GKE)**, complete with a comprehensive monitoring stack using **Prometheus** and **Grafana**. This setup demonstrates a multi-tier web application deployment and its observability in a cloud environment.

**Key Components:**
* **Google Kubernetes Engine (GKE):** Google Cloud's managed Kubernetes service.
* **Guestbook Application:**
    * **Redis Master:** Stores Guestbook entries.
    * **Redis Slaves:** Replicas for read operations.
    * **PHP Frontend:** The web interface for user interaction.
* **Prometheus:** An open-source monitoring system for collecting metrics.
* **Grafana:** An open-source platform for analytics and metric visualization.

---

## 📋 Table of Contents

1.  [System Architecture](#1-system-architecture)
2.  [Prerequisites](#2-prerequisites)
3.  [Repository Structure](#3-repository-structure)
4.  [Deployment Guide](#4-deployment-guide)
    * 4.1. GCP Project Setup
    * 4.2. GKE Cluster Creation
    * 4.3. Guestbook Application Deployment
    * 4.4. Prometheus & Grafana Deployment
5.  [Accessing the Applications](#5-accessing-the-applications)
    * 5.1. Accessing the Guestbook
    * 5.2. Accessing Grafana
6.  [Cost Optimization (Free Tier / Development)](#6-cost-optimization-free-tier--development)
7.  [Resource Cleanup](#7-resource-cleanup)
8.  [Important Notes & Considerations](#8-important-notes--considerations)
9.  [References & Useful Resources](#9-references--useful-resources)
10. [License](#10-license)

---

## 1. 🏗️ System Architecture

This diagram illustrates the high-level interaction between the components deployed in your GKE cluster:

```bash
[User]
|
V
[Google Cloud Load Balancer]
|
V
[GKE Frontend Pods (PHP)] ---read---> [Redis Slaves Pods]
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
* ** Intallation process:

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
 
     * Install google services to interact with GKE
       gcloud services enable container.googleapis.com compute.googleapis.com

       ![image](https://github.com/user-attachments/assets/27fd85be-be4b-491d-a772-7735668f7854)

* **Note:** Ensure your GCP account has the necessary IAM permissions in your project to enable APIs

   ![image](https://github.com/user-attachments/assets/54f2dff5-62f3-466d-a47b-1fc8363e7ed5)
      

* **Neccessary Tools     
* **kubectl:** The Kubernetes command-line tool. << Interconnection between cli and GKE to create and admin deployments
    * [kubectl Installation Guide](https://kubernetes.io/docs/tasks/tools/install-kubectl/) (often included with gcloud SDK)
* **Helm (v3+):** The Kubernetes package manager.
    * [Helm Installation Guide](https://helm.sh/docs/intro/install/)
* **Git:** For cloning this repository.(Optional) <<<< This help us to copy the repository in our local machine in case is necessary to edit some files
    * [Git Installation Guide](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)

* Commands to verify Prerequisites
```bash
  * gcloud version
  * gcloud projects list
  * kubectl version
  * helm version
```

Expected result:

![image](https://github.com/user-attachments/assets/575c7f6a-5e9e-407e-b86b-73e83ce81fda)

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

1.  **Enable Required APIs:**
    ```bash
    gcloud services enable container.googleapis.com compute.googleapis.com
    ```
    * **Note:** Ensure your GCP account has the necessary IAM permissions (e.g., `Service Usage Admin` or `Owner`) in your project to enable APIs.

### 4.2. GKE Cluster Creation

To optimize costs, we'll create a small cluster suitable for testing.

1.  **Create the GKE Cluster:**
    ```bash
    gcloud container clusters create guestbook-cluster \
      --project YOUR_PROJECT_ID \ # Replace YOUR_PROJECT_ID
      --region us-central1 \ # Or your preferred region with good quota
      --num-nodes 1 \       # Minimum nodes to save costs
      --machine-type e2-small \ # Economical machine type
      --disk-size 20GB       # Node boot disk size. Adjust if you hit quota issues.
    ```

2.  **Configure `kubectl` to access your cluster:**
    ```bash
    gcloud container clusters get-credentials guestbook-cluster --region us-central1 --project YOUR_PROJECT_ID
    ```
    * **Important:** If you receive a critical message about `gke-gcloud-auth-plugin` not being found or executable, you might need to install it manually. See the [References](#9-references--useful-resources) section for the official installation guide.

3.  **Verify cluster nodes:**
    ```bash
    kubectl get nodes
    ```

### 4.3. Guestbook Application Deployment

First, ensure you have the Guestbook YAML files locally. If you're building this repo by copying files directly to GitHub, you'll just be committing these from your local copy.

1.  **Deploy Redis Master:**
    ```bash
    kubectl apply -f guestbook/redis-master-deployment.yaml
    kubectl apply -f guestbook/redis-master-service.yaml
    ```
2.  **Deploy Redis Slaves:**
    * **Optimization Tip:** For free tier usage, consider editing `guestbook/redis-slave-deployment.yaml` to set `replicas: 1` if it's not already.
    ```bash
    kubectl apply -f guestbook/redis-slave-deployment.yaml
    kubectl apply -f guestbook/redis-slave-service.yaml
    ```
3.  **Deploy Frontend (PHP Guestbook):**
    * **Optimization Tip:** Edit `guestbook/frontend-deployment.yaml` to set `replicas: 1`.
    * **Crucial:** Ensure `guestbook/frontend-service.yaml` has `type: LoadBalancer` to expose the application to the internet. **You'll need to manually edit this file in GitHub's interface or after cloning locally.**

    ```yaml
    # Excerpt from guestbook/frontend-service.yaml
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

### 4.4. Prometheus & Grafana Deployment

1.  **Add the Helm Repository:**
    ```bash
    helm repo add prometheus-community [https://prometheus-community.github.io/helm-charts](https://prometheus-community.github.io/helm-charts)
    helm repo update
    ```
2.  **Create a Dedicated Namespace:**
    ```bash
    kubectl create namespace monitoring
    ```
3.  **Prepare the `monitoring/prometheus-grafana-values.yaml` file:**
    Open the `monitoring/prometheus-grafana-values.yaml` file. This file contains configurations to optimize resource consumption (e.g., disabling Alertmanager, adjusting CPU/Memory requests/limits).
    * **CHANGE THE GRAFANA ADMIN PASSWORD!**
        Find `adminPassword: "yourStrongPassword"` and set a **strong, unique password**.
    * **Grafana Service Type:** By default, it's set to `type: LoadBalancer` for easy external access. If you prefer `NodePort` for cost savings, adjust it here.

    ```yaml
    # Excerpt from monitoring/prometheus-grafana-values.yaml
    grafana:
      enabled: true
      adminPassword: "yourStrongPassword" # <--- CHANGE THIS!
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

5.  **Verify monitoring deployment:**
    ```bash
    kubectl get pods -n monitoring
    kubectl get services -n monitoring
    ```

---

## 5. 🌐 Accessing the Applications

### 5.1. Accessing the Guestbook

1.  **Get the Frontend's External IP:**
    ```bash
    kubectl get service frontend
    ```
    Wait until the `EXTERNAL-IP` column shows a public IP address (this might take a few minutes).

2.  **Open in Browser:**
    Navigate to `http://<GUESTBOOK_EXTERNAL_IP>` in your web browser.

### 5.2. Accessing Grafana

1.  **Get Grafana's External IP:**
    ```bash
    kubectl get service prometheus-grafana -n monitoring
    ```
    Wait until the `EXTERNAL-IP` column shows a public IP address.

2.  **Get Grafana Admin Password:**
    The username is `admin`. The password is what you set in `monitoring/prometheus-grafana-values.yaml`. If you need to retrieve it later:
    ```bash
    kubectl get secret --namespace monitoring prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
    ```

3.  **Open in Browser:**
    Navigate to `http://<GRAFANA_EXTERNAL_IP>` in your web browser and log in with the credentials.

---
