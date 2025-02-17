# Azure AKS Cluster Setup

This repository will guide you through the preparation process of building an AKS cluster using the Azure Cloud Shell.

## Getting Started with Azure Cloud Shell

The following are the basic requirements to **start** the setup.

* Azure Account [Azure Portal](https://portal.azure.com)
* Git [Git SCM](https://git-scm.com/downloads)
* Azure Cloud Shell [https://shell.azure.com](https://shell.azure.com)
* Azure AKS Cluster 

# Instructions

1. Login to Azure Portal at http://portal.azure.com.
2. Open the Azure Cloud Shell and choose Bash Shell (do not choose Powershell)

   ![214944180-0b72595f-b58d-445d-9bde-2530bd491ace](https://github.com/user-attachments/assets/bdb8c952-7053-46f6-bd86-fdb8def3596c)

3. The first time Cloud Shell is started will require you to create a storage account.

   > Note: In the cloud shell, you are automatically logged into your Azure subscription.

4. [Optional] If you have more than one Azure subscription, ensure you are using the one you want to deploy AKS to.

   View subscriptions
   ```bash
   az account list
   ```

   Verify selected subscription
   ```bash
   az account show
   ```

   Set correct subscription (if needed)
   ```bash
   az account set --subscription <subscription_id>
   ```
   
   Verify correct subscription is now set
   ```bash
   az account show
   ```

5. Configure the kubectl autocomplete.

   ```bash
   source <(kubectl completion bash) &&  source /usr/share/bash-completion/bash_completion
   echo "source <(kubectl completion bash)" >> ~/.bashrc
   echo "source /usr/share/bash-completion/bash_completion" >> ~/.bashrc
   ```

   You can also use a shorthand alias for kubectl that also works with completion:

   ```bash
   alias k=kubectl
   complete -o default -F __start_kubectl k
   echo "alias k=kubectl"  >> ~/.bashrc
   echo "complete -o default -F __start_kubectl k" >> ~/.bashrc
   ```

6. [Optional] Instal k9s, if you like it.

   ```bash
   curl --silent --location "https://github.com/derailed/k9s/releases/download/v0.32.7/k9s_Linux_amd64.tar.gz" | tar xz -C /tmp
   mkdir -p ~/.local/bin
   mv /tmp/k9s ~/.local/bin 
   k9s version
   ```

## Deploy an Azure AKS Cluster

1. Define the environment variables to be used by the resources definition.

   > **NOTE**: In the following section, we'll create some environment variables. If your terminal session restarts, you may need to reset these variables. You can do that using the following command:
   >
   > ```console
   > source ~/aksvars.env
   > ```

   ```bash
   export RESOURCE_GROUP=rmartins
   export CLUSTERNAME=aks-cluster
   export LOCATION=westus2
   # Persist for later sessions in case of disconnection.
   echo export RESOURCE_GROUP=$RESOURCE_GROUP > ~/aksvars.env
   echo export CLUSTERNAME=$CLUSTERNAME >> ~/aksvars.env
   echo export LOCATION=$LOCATION >> ~/aksvars.env
   ```

2. If not created, create the Resource Group in the desired Region.
   
   ```bash
   az group create \
     --name $RESOURCE_GROUP \
     --location $LOCATION
   ```
   
3. Create the AKS cluster without a network plugin.
   
   ```bash
   az aks create \
     --resource-group $RESOURCE_GROUP \
     --name $CLUSTERNAME \
     --kubernetes-version 1.30 \
     --location $LOCATION \
     --node-count 2 \
     --node-vm-size Standard_B2ms \
     --max-pods 100 \
     --generate-ssh-keys \
     --network-plugin azure
   ```

4. Verify your cluster status. The `ProvisioningState` should be `Succeeded`

   ```bash
   az aks list -o table | grep $CLUSTERNAME
   ```
 
   You may get an output like the following:

   <pre>
   aks-cluster       westus2     rmartins              1.30                 1.30.7                      Succeeded            aks-cluste-rmartins-03cfb8-3lfxk7df.hcp.westus2.azmk8s.io
   </pre>

5. Get the credentials to connect to the cluster.
   
   ```bash
   az aks get-credentials --resource-group $RESOURCE_GROUP --name $CLUSTERNAME
   ```

6. Verify you have API access to your new AKS cluster

   ```bash
   kubectl get nodes
   ```

   The output will be something similar to the this:

   <pre>

   NAME                                STATUS   ROLES    AGE     VERSION
   aks-nodepool1-96098601-vmss000000   Ready    <none>   5m46s   v1.30.7
   aks-nodepool1-96098601-vmss000001   Ready    <none>   5m53s   v1.30.7
   </pre>

   To see more details about your cluster:

   ```bash
    kubectl cluster-info
   ```

   The output will be something similar to the this:
   <pre>
   Kubernetes control plane is running at https://aks-cluste-rmartins-03cfb8-3lfxk7df.hcp.westus2.azmk8s.io:443
   CoreDNS is running at https://aks-cluste-rmartins-03cfb8-3lfxk7df.hcp.westus2.azmk8s.io:443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
   Metrics-server is running at https://aks-cluste-rmartins-03cfb8-3lfxk7df.hcp.westus2.azmk8s.io:443/api/v1/namespaces/kube-system/services/https:metrics-server:/proxy

   To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
   </pre>

   You should now have a Kubernetes cluster running with 2 nodes. You do not see the master servers for the cluster because these are managed by Microsoft. The Control Plane services which manage the Kubernetes cluster such as scheduling, API access, configuration data store and object controllers are all provided as services to the nodes.

7. Verify the network settings.
   
   ```bash
   az aks show --resource-group $RESOURCE_GROUP --name $CLUSTERNAME --query 'networkProfile'
   ```

   You should see "networkPlugin": "azure" and "networkPolicy": null (networkPolicy will just not show if it is null).

8. Verify the transparent mode by running the following command in one node

   ```bash
   VMSSGROUP=$(az vmss list --output table | grep -i $RESOURCE_GROUP | grep -i $CLUSTERNAME | awk -F ' ' '{print $2}')
   VMSSNAME=$(az vmss list --output table | grep -i $RESOURCE_GROUP | grep -i $CLUSTERNAME | awk -F ' ' '{print $1}')
   az vmss run-command invoke -g $VMSSGROUP -n $VMSSNAME --scripts "cat /etc/cni/net.d/*" --command-id RunShellScript --instance-id 0 --query 'value[0].message' --output table
   ```
   
   > output should contain "mode": "transparent"

9. Stop the Cluster.

   ```bash
   az aks stop --resource-group $RESOURCE_GROUP --name $CLUSTERNAME
   ```

10. To start your cluster.

    ```bash
    az aks start --resource-group $RESOURCE_GROUP --name $CLUSTERNAME
    ```

## Clean up

1. Delete the AKS cluster.

   ```bash
   az aks delete \
     --resource-group $RESOURCE_GROUP \
     --name $CLUSTERNAME
   ```

2. Delete the resource group.

   ```bash
   az group delete \
     --name $RESOURCE_GROUP
   ```

3. Delete environment variables backup file.

   ```bash
   rm aksvars.env
   ```
