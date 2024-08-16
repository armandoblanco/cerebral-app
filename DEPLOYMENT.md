# Cerebral Implementation Guide

This comprehensive guide details the steps to install and configure Cerebral, a smart assistant designed to enhance operational efficiency at Contoso Motors through the use of Generative AI.

## Step 1 - Setting Up the Infrastructure

### Prepare Your Azure Arc-Enabled Kubernetes Cluster on Ubuntu

#### Requirements:
- An Ubuntu VM with internet access
- User with sudo privileges

#### Installation Steps:

1. **Install Necessary Tools:**
   - Update and install `curl`:
     ```bash
     sudo apt update && sudo apt install curl -y
     ```
   - Install Azure CLI:
     ```bash
     curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
     ```
   - Add the Azure IoT operations extension:
     ```bash
     az extension add --upgrade --name azure-iot-ops
     ```

2. **Install and Configure K3s:**
   - Execute:
     ```bash
     curl -sfL https://get.k3s.io | sh –
     ```
   - Configure kubectl:
     ```bash
     mkdir ~/.kube
     sudo KUBECONFIG=~/.kube/config:/etc/rancher/k3s/k3s.yaml kubectl config view --flatten > ~/.kube/merged
     mv ~/.kube/merged ~/.kube/config
     chmod 0600 ~/.kube/config
     export KUBECONFIG=~/.kube/config
     kubectl config use-context default
     ```

3. **Optimize System Performance:**
   - Adjust user watch limits:
     ```bash
     echo fs.inotify.max_user_instances=8192 | sudo tee -a /etc/sysctl.conf
     echo fs.inotify.max_user_watches=524288 | sudo tee -a /etc/sysctl.conf
     sudo sysctl -p
     ```

4. **Connect Your Cluster to Azure Arc:**
   - Authenticate and set Azure context:
     ```bash
     az login
     az account set --subscription "<Your_Subscription_ID>"
     ```
   - Create a resource group and connect the Kubernetes cluster:
     ```bash
     az group create --location "<Azure_Region>" --name "<Resource_Group_Name>"
     az connectedk8s connect --name "<Cluster_Name>" --resource-group "<Resource_Group_Name>"
     ```

5. **Install Azure IoT Operations:**
   - Verify and install Azure IoT Operations on your cluster:
     ```bash
     az iot ops verify-host
     az iot ops init --name "<IoT_Ops_Instance_Name>" --resource-group "<Resource_Group_Name>" --location "<Azure_Region>"
     ```

6. **Create an Azure Key Vault:**
   - Ensure that the Key Vault uses the Vault access policy:
     ```bash
     az keyvault create --name "<Key_Vault_Name>" --resource-group "<Resource_Group_Name>" --location "<Azure_Region>" --enable-rbac-authorization false
     ```

## Step 2 - Deploy Cerebral

### Deploy Core Components

1. **Namespace and Persistent Data Handling:**
   - Create namespace and configure persistent storage for InfluxDB:
     ```bash
     kubectl apply -f https://raw.githubusercontent.com/armandoblanco/cerebral-app/main/deployment/cerebral-ns.yaml
     sudo mkdir /var/lib/influxdb2 && sudo chmod 777 /var/lib/influxdb2
     kubectl apply -f https://raw.githubusercontent.com/armandoblanco/cerebral-app/main/deployment/influxdb.yaml
     ```

2. **Install Redis Cache:**
   - Deploy Redis for session management:
     ```bash
     kubectl apply -f https://raw.githubusercontent.com/armandoblanco/cerebral-app/main/deployment/redis.yaml
     ```

3. **Configure and Run Cerebral:**
   - Configure and deploy the Cerebral application:
     ```bash
     wget https://raw.githubusercontent.com/armandoblanco/cerebral-app/main/deployment/cerebral.yaml
     nano cerebral.yaml  # Adjust configuration as necessary
     kubectl apply -f cerebral.yaml
     ```

### Validation and Testing

- Confirm all components are operational:
  ```bash
  kubectl get all -n cerebral
