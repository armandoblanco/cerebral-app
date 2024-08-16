# Cerebral Implementation Guide

## Introduction

## Introduction to Cerebral

Cerebral is an innovative smart assistant tailored for industrial applications, specifically designed to enhance operational efficiency at Contoso Motors. This solution leverages advanced Generative AI technology to interact with factory workers using natural language, making it intuitive and user-friendly. Cerebral’s core aim is to streamline decision-making processes and reduce downtime by providing immediate access to critical information and operational insights.

### Key Features and Benefits

- **Natural Language Processing**: At the heart of Cerebral is its ability to understand and process queries in natural language. This allows plant workers to ask complex questions about machine operations, maintenance schedules, or troubleshooting steps as if they were speaking to a human expert.

- **Dynamic Information Access**: Depending on the nature of the query, Cerebral can search through extensive databases of manuals and guidelines to provide troubleshooting assistance or access real-time data directly from the production line’s equipment and sensors. This ensures that the information provided is not only accurate but also timely and relevant.

- **Hybrid Model Utilization**: The architecture of Cerebral incorporates both cloud-based and on-premises components, including the use of Open AI for cloud-based computations and an on-premises SLM for handling sensitive data locally. This hybrid approach ensures optimal balance between performance and security, adhering to industry best practices for data governance. 

- **Enhanced Decision Making**: Through its integration with advanced data analytics platforms like Azure Data Explorer, Cerebral offers powerful visualization tools that help users identify trends, analyze performance metrics, and make well-informed decisions faster than ever before.

- **Customizable and Scalable**: The system is designed to be flexible, supporting modifications and enhancements to meet the evolving needs of Contoso Motors. It can scale across different departments and adapt to various industrial environments without significant alterations to the core system.

### Target Audience

Cerebral is specifically developed for operational technology professionals including mechanics, maintenance staff, and production line managers. It simplifies their daily tasks by providing a seamless interface to query operational data, access procedural documents, and gain insights into machinery health and performance.

This smart assistant is not just a tool but a part of the team, designed to work alongside factory personnel to enhance productivity and ensure that the manufacturing processes at Contoso Motors are as efficient as possible.

By integrating Cerebral, Contoso Motors aims to set a new standard in industrial operations, focusing on connectivity, speed, and intelligence. The upcoming sections will detail the technical architecture, setup instructions, and operational guidelines to fully leverage Cerebral’s capabilities in a manufacturing setting.


## Architecture

## Cerebral Architecture Overview

The architecture of Cerebral integrates various components to provide a robust solution for real-time data processing and query handling within an industrial setting. The system is designed to be deployed on a factory-floor located, Arc-enabled AKS Edge Essentials cluster, ensuring that both data security and processing efficiency are optimized.

### Key Components

1. **OT Frontline Worker Interface**:
   - This is the primary user interface where operational technology (OT) frontline workers, such as mechanics, maintenance personnel, or plan managers, interact with the Cerebral system. Users can input their queries in natural language, which are then processed by the system to fetch relevant information or perform actions.

2. **Redis Cache**:
   - Utilized as a caching layer to store temporary data which may include session states, user preferences, and frequently accessed data to speed up response times.

3. **Web Application**:
   - Hosts the user interface and the agent responsible for classifying questions based on the input received from the OT frontline worker.

4. **Classify Agent**:
   - Analyzes the questions to determine the type of query and routes it to the appropriate processing path, either pulling data from real-time systems or fetching documents.

5. **Query Processing Orchestrator**:
   - Manages the workflow of data queries and document retrievals, ensuring that requests are processed efficiently and correctly routed to either InfluxDB for data-related queries or the Chroma vector database for document retrievals.

6. **Azure OpenAI**:
   - Provides the AI and machine learning backbone, analyzing queries and generating responses that are contextually aware and relevant to the user’s needs.

7. **InfluxDB (Time Series Data)**:
   - A database optimized for storing and retrieving time-series data from various equipment and sensors on the production line.

8. **Chroma Vector Database**:
   - Stores and manages access to manuals and troubleshooting guides which are used to answer queries related to equipment maintenance and other operational procedures.

9. **SLM/LLM Model “Phi-2”**:
   - A sophisticated language model that helps in interpreting complex technical queries and generating accurate responses based on the contextual understanding of the industry-specific data.

10. **Assembly Line Simulator**:
    - Simulates data from the production line, which can be used for testing and demonstration purposes without the need to access the actual production environment.

11. **Azure IoT Operations and MQTT Broker**:
    - Manages device communication and data flow between the on-premises infrastructure and Azure services, ensuring secure and reliable data handling.


### Communication Flow

- Data flows through the system starting from the user query input through the OT frontline worker interface.
- The classify agent determines the nature of the query and passes it to the query processing orchestrator.
- Based on the query type, it might interact with Azure OpenAI for AI-driven insights or direct the query to either InfluxDB for real-time data or the Chroma vector database for document retrieval.
- The results are then presented back to the user through the web application, potentially utilizing ADX dashboards for enhanced data visualization.

This modular yet integrated architecture allows Cerebral to offer a flexible, scalable solution adaptable to various industrial environments, enhancing operational efficiency through AI-driven automation and real-time data processing.

![Cerebral Architecture Diagram](/resources/images/architecture.png)

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
