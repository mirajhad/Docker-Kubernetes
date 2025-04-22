# Deploy an Azure Kubernetes Service (AKS) cluster using Azure CLI
Login to specific tenant (from Entra ID)
```   
az login --tenant <TENANT_ID>
    az login --tenant 8ae4526f-98aa-4bcc-b9c8-abd102c7d981
```

Create resource group
```
az group create --name ecommerce-resource-group --location "South India"
```

Create container registry
```
az acr create --resource-group ecommerce-resource-group --name mirajecommerceregistry --sku Basic
```

Register Microsoft.Insights Resource Provider, which is required for Cluster monitoring:
```
az provider register --namespace Microsoft.Insights 
az provider register --namespace Microsoft.OperationalInsights   
az provider register --namespace Microsoft.ContainerService   
az provider register --namespace Microsoft.Network 
az provider register --namespace Microsoft.Compute   
az provider register --namespace Microsoft.OperationsManagement   
az provider register --namespace Microsoft.Authorization   
az provider register --namespace Microsoft.Storage 
```
Microsoft.Insights: Enables monitoring and diagnostics for Azure resources.
Microsoft.OperationalInsights: Supports Log Analytics for operational data monitoring.
Microsoft.ContainerService: Manages Azure Kubernetes Service (AKS) and container workloads.
Microsoft.Network: Provides network-related services like VNETs, Load Balancers, and VPNs.
Microsoft.Compute: Manages virtual machines, scale sets, and related compute resources.
Microsoft.OperationsManagement: Handles operations management solutions, like Azure Automation.
Microsoft.Authorization: Manages access control (RBAC) and policy settings for resources.
Microsoft.Storage: Manages Azure Storage services like blobs, files, and queues.



Create AKS cluster:
```
az aks create --resource-group ecommerce-resource-group --name ecommerce-aks-cluster --node-count 1 --node-vm-size Standard_B2s --enable-addons monitoring --generate-ssh-keys
```
--node-count 1

Node Count: Specifies the number of nodes (virtual machines) in your AKS cluster.
Nodes: Each node is a VM that runs your Kubernetes workloads (containers).
In this command, --node-count 1 means the cluster will have 1 nodes. You can add more; for example "5" if you need more resources in real world projects. You can update it later, if needed.

Default: Standard_B2s:
vCPUs: 2
Memory: 8 GB
Temporary Storage: 4 GB
Use Case: Cost-effective for burstable workloads where CPU usage is not consistent but needs to handle spikes in demand.

Other Options:
Standard_DS2_v2: A more powerful VM, suitable for production workloads.
Standard_E8s_v3: High memory VMs for memory-intensive application

--enable-addons monitoring

Enable Addons: This flag is used to enable additional features in your AKS cluster.

Monitoring: In this case, monitoring refers to enabling Azure Monitor for containers, which provides insights and monitoring capabilities for your Kubernetes cluster.

Even for learning purposes, it's useful to enable monitoring so you can observe the performance and behavior of your applications and the Kubernetes cluster itself.


--enable-managed-identity: 
This enables a system-assigned managed identity for the AKS cluster, which is useful for securely accessing Azure resources in your AKS cluster.


--generate-ssh-keys

SSH Keys: Secure Shell (SSH) keys are used to securely access the virtual machines (nodes) in your cluster.

If you need to troubleshoot or manage the individual nodes (VMs), SSH keys allow you to securely connect to them.

SSH keys are a way to securely access remote nodes (VMs) without needing to type in a password each time. They work using a pair of keys:   

Public Key: This key is placed on the remote server you want to access. It's like leaving a padlock on the server's door.   
Private Key: This key stays on your computer and is kept secret. It's like the key to the padlock you left on the server.

When you want to connect, your computer uses your private key to prove it has the right to unlock the padlock (the public key) on the server. This allows you to log in to the node (VM) without a password.   



Enable "kubectl" command:
```
az aks get-credentials --resource-group ecommerce-resource-group --name ecommerce-aks-cluster
```
This command fetches the credentials and configuration details for your AKS cluster and updates your local kubeconfig file. 
It allows your local machine to connect to the AKS cluster and use kubectl to manage it. 
By running this, you enable your local Kubernetes client to interact with your cluster, making it possible to deploy and manage applications from your local setup.


Get cluster info:
```
kubectl cluster-info
```    
This command provides information about the Kubernetes cluster you are currently connected to. 


kubectl Quick Reference: https://kubernetes.io/docs/reference/kubectl/quick-reference/

    kubectl [or] kubectl --help

Get current cluster name [Optional]:
```
kubectl config current-context
```



## Kubectl Basic Command

Get list of nodes in your cluster:
```
kubectl get nodes
```

This should return a list of nodes in your AKS cluster.
We've created the cluster with two worker nodes.


Get list of namespaces in your cluster:
```
kubectl get namespaces
```

Lists all namespaces. Namespaces provide a way to divide cluster resources between multiple users or teams.
You will have to create one namespace for each set of resources (such as Pods) that need to be managed by a user or a team.

Namespaces in Kubernetes are a way to divide a single cluster into multiple virtual clusters. 
    
They help organize and manage resources, allowing different teams or applications to share the same cluster without interfering with each other. 
    
By using namespaces, you can isolate resources like pods and services, set resource quotas, and apply policies separately for each namespace. This makes it easier to manage and secure resources within a large, shared Kubernetes environment.

## Deployment Commands
```
az login
az acr login --name mirajecommerceregistry
```

TO DO: Create the deployment file (deployment.yaml)
### Deployment yaml file
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demowebapp-deployment
  labels:
    app: demo-webapi-app
    environment: development
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demowebapp
  template:
    metadata:
      labels:
        app: demowebapp
    spec:
        containers:
            - name: demowebapp-container
              image: mirajecommerceregistry.azurecr.io/demowebapp:latest
              ports:
               - containerPort: 80
              env:
               - name: ASPNETCORE_ENVIRONMENT
                 value: Development
```

```
cd c:\azure\DemoWebApplicationSolution

docker build -t demowebapp:latest -f ./DemoWebApplication/Dockerfile .

docker tag demowebapp:latest harshaecommerceregistry.azurecr.io/demowebapp:latest

docker push mirajecommerceregistry.azurecr.io/demowebapp:latest
```

Attach ACR to the cluster (Grant the AKS managed identity the AcrPull role on the ACR):
```    
az aks update --resource-group ecommerce-resource-group --name ecommerce-aks-cluster --attach-acr mirajecommerceregistry
```

Apply the deployment yaml file:
```
cd DemoWebApplication
kubectl apply -f deployment.yaml
```

Get list of deployments in the current namespace:
```
kubectl get deployments
```
Lists all deployments in the current namespace.
A deployment in Kubernetes is a resource that manages the creation, updating, and scaling of a set of replicas of a pod. It ensures that the desired number of pod replicas are running and provides rolling updates and rollback capabilities to maintain application availability and consistency.


Get deployment details:
```
kubectl describe deployment demowebapp-deployment
```

Get list of pods in your cluster:
```
kubectl get pods
```

Get list of pods of a specific deployment:
```
kubectl get pods --selector=app=demowebapp
```

Get list of containers in a pod:
```
kubectl describe pod demowebapp-deployment-856cff9945-5k6p8
```

Get logs of first container of a pod:
```
kubectl logs demowebapp-deployment-856cff9945-5k6p8
```

Get logs of specific container in a pod:
```
kubectl logs demowebapp-deployment-856cff9945-5k6p8 -c demowebapp-container
```

Delete a deployment:
```
kubectl delete deployment demowebapp-deployment --namespace default
```


[Not necessary]
Add managed-identity on existing cluster:
```
az aks update --resource-group demo-resource-group --name ecommerce-aks-cluster --enable-managed-identity
```

# Service Commands

Apply service:
```
kubectl apply -f service.yaml
```
//TODO:creating service.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: demowebapp-service
spec:
  type: LoadBalancer
  selector:
    app: demowebapp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080

```

Get list of services in your cluster:
```
kubectl get services
```
Services provide a stable endpoint for accessing pods.
Get service details:
```
kubectl get service demowebapp-service
```

Get more service details:
```
kubectl describe service demowebapp-service
```







