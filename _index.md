# Cerebral Implementation Guide

## Introduction

Cerebral is a smart assistant designed to enhance operational efficiency by providing real-time troubleshooting and data analysis capabilities. This guide will walk you through the steps to deploy Cerebral on a Kubernetes cluster, including both Azure OpenAI integration and RAG (Retrieval-Augmented Generation) configuration at the Edge.

## Architecture

![Cerebral Architecture](./images/architecture.png)

## Solution Overview

The solution is divided into two main components:
1. **Azure OpenAI Implementation:** This involves setting up Cerebral to leverage Azure OpenAI for natural language processing and query classification.
2. **RAG at the Edge:** This involves configuring Cerebral to use Chroma as a vector database and PHI-2 for on-premises language processing, allowing data processing to occur at the edge for faster response times.

## Prerequisites

Before starting, ensure you have the following:
- **Azure Subscription**: An active Azure subscription.
- **Azure CLI**: Installed and configured on your machine.
- **Visual Studio Code**: With Azure extensions for easier management.
- **K3S**: A lightweight Kubernetes distribution.
- **Cluster Enrolled in Azure Arc**: Your K3S cluster should be enrolled in Azure Arc.
- **Azure IoT Operations**: Set up for managing IoT devices and edge deployments.
- **Security Requirements**: Proper security configurations, including RBAC and network policies.

## Solution Build Steps

### Step 1 - Building an Ubuntu VM running Azure IoT Operation

1. **Prepare Your Azure Arc-enabled Kubernetes Cluster on Ubuntu:**
   - Install `curl`:
     ```bash
     sudo apt install curl -y
     ```
   - Install Azure CLI:
     ```bash
     curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
     ```
   - Install Azure IoT Operations extension:
     ```bash
     az extension add --upgrade --name azure-iot-ops
     ```
   - Install K3S:
     ```bash
     curl -sfL https://get.k3s.io | sh –
     ```

2. **Set Up K3S Configuration:**
   - Create K3S configuration:
     ```bash
     mkdir ~/.kube
     sudo KUBECONFIG=~/.kube/config:/etc/rancher/k3s/k3s.yaml kubectl config view --flatten > ~/.kube/merged
     mv ~/.kube/merged ~/.kube/config
     chmod  0600 ~/.kube/config
     export KUBECONFIG=~/.kube/config
     kubectl config use-context default
     ```
   - Increase user watch/instance limits:
     ```bash
     echo fs.inotify.max_user_instances=8192 | sudo tee -a /etc/sysctl.conf
     echo fs.inotify.max_user_watches=524288 | sudo tee -a /etc/sysctl.conf
     sudo sysctl -p
     ```
   - Increase file descriptor limit:
     ```bash
     echo fs.file-max = 100000 | sudo tee -a /etc/sysctl.conf
     sudo sysctl -p
     ```

3. **Connect Your Cluster to Azure Arc:**
   - Login to Azure:
     ```bash
     az login
     ```
   - Set environment variables:
     ```bash
     export SUBSCRIPTION_ID=<YOUR_SUBSCRIPTION_ID>
     export LOCATION=<YOUR_REGION>
     export RESOURCE_GROUP=<YOUR_RESOURCE_GROUP>
     export CLUSTER_NAME=<YOUR_CLUSTER_NAME>
     export KV_NAME=<YOUR_KEY_VAULT_NAME>
     export INSTANCE_NAME=<YOUR_INSTANCE_NAME>
     ```
   - Set Azure subscription context:
     ```bash
     az account set -s $SUBSCRIPTION_ID
     ```
   - Register required resource providers:
     ```bash
     az provider register -n "Microsoft.ExtendedLocation"
     az provider register -n "Microsoft.Kubernetes"
     az provider register -n "Microsoft.KubernetesConfiguration"
     az provider register -n "Microsoft.IoTOperationsOrchestrator"
     az provider register -n "Microsoft.IoTOperations"
     az provider register -n "Microsoft.DeviceRegistry"
     ```
   - Create a resource group:
     ```bash
     az group create --location $LOCATION --resource-group $RESOURCE_GROUP --subscription $SUBSCRIPTION_ID
     ```
   - Connect Kubernetes cluster to Azure Arc:
     ```bash
     az connectedk8s connect -n $CLUSTER_NAME -l $LOCATION -g $RESOURCE_GROUP --subscription $SUBSCRIPTION_ID
     ```
   - Get `objectId` of Microsoft Entra ID application:
     ```bash
     export OBJECT_ID=$(az ad sp show --id bc313c14-388c-4e7d-a58e-70017303ee3b --query id -o tsv)
     ```
   - Enable custom location support:
     ```bash
     az connectedk8s enable-features -n $CLUSTER_NAME -g $RESOURCE_GROUP --custom-locations-oid $OBJECT_ID --features cluster-connect custom-locations
     ```
   - Verify cluster readiness for Azure IoT Operations:
     ```bash
     az iot ops verify-host
     ```
   - Create an Azure Key Vault:
     ```bash
     az keyvault create --enable-rbac-authorization false --name $KV_NAME --resource-group $RESOURCE_GROUP
     ```

4. **Deploy Azure IoT Operations:**
   - Verify cluster host configuration:
     ```bash
     az iot ops verify-host
     ```
   - Deploy Azure IoT Operations:
     ```bash
     az iot ops init --subscription $SUBSCRIPTION_ID -g $RESOURCE_GROUP --cluster $CLUSTER_NAME --custom-location testscriptscluster-cl-4694 -n $INSTANCE_NAME --broker broker --kv-id /subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.KeyVault/vaults/$KV_NAME --add-insecure-listener --simulate-plc
     ```

### Step 2 - Install Cerebral

1. **Deploy Namespace, InfluxDB, Simulator, and Redis:**
   - Create a folder for Cerebral configuration files:
     ```bash
     mkdir cerebral
     cd cerebral
     ```
   - Apply the Cerebral namespace:
     ```bash
     kubectl apply -f https://raw.githubusercontent.com/armandoblanco/cerebral-app/main/deployment/cerebral-ns.yaml
     ```
   - Create a directory for persistent InfluxDB data:
     ```bash
     sudo mkdir /var/lib/influxdb2
     sudo chmod 777 /var/lib/influxdb2
     ```
   - Deploy InfluxDB:
     ```bash
     kubectl apply -f https://raw.githubusercontent.com/armandoblanco/cerebral-app/main/deployment/influxdb.yaml
     ```
   - Configure InfluxDB:
     ```bash
     kubectl apply -f https://raw.githubusercontent.com/armandoblanco/cerebral-app/main/deployment/influxdb-setup.yaml
     ```
   - Deploy the data simulator:
     ```bash
     kubectl apply -f https://raw.githubusercontent.com/armandoblanco/cerebral-app/main/deployment/cerebral-simulator.yaml
     ```
   - Validate the implementation:
     ```bash
     kubectl get all -n cerebral
     ```

2. **Access InfluxDB:**
   - Use a web browser to connect to the public IP of the InfluxDB service to access its interface. Validate that there is a bucket named `manufactura` and that it contains a measurement called `assembly line` with values.

3. **Install Redis:**
   - Deploy Redis to store user sessions and conversation history:
     ```bash
     kubectl apply -f https://raw.githubusercontent.com/armandoblanco/cerebral-app/main/deployment/redis.yaml
     ```

4. **Deploy Cerebral Application:**
   - Create an Azure OpenAI service in your subscription and obtain the key and endpoint for use in the application configuration.
   - Download the Cerebral application deployment file:
     ```bash
     wget https://raw.githubusercontent.com/armandoblanco/cerebral-app/main/deployment/cerebral.yaml
     ```
   - Edit the file with your Azure OpenAI instance details:
     ```bash
     nano cerebral.yaml
     ```
   - Apply the changes and deploy the application:
     ```bash
     kubectl apply -f cerebral.yaml
     ```

5. **Verify All Components:**
   - Ensure that all components are functioning correctly by checking the pods and services.
   

### Conclusion

By following these steps, you should have a fully functioning Cerebral implementation integrated with Azure IoT Operations and ready for real-time data analysis and troubleshooting. This setup supports both Azure OpenAI and on-premises RAG processing, offering flexibility and high performance in diverse operational environments.
