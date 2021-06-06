# variables

    aksClusterGroupName="aciburstdemo" # Name of resource group for the AKS cluster
    aksName="Arcfork8s-demo" # Name of the AKS cluster
    resourceLocation="westeurope" # "eastus" or "westeurope"
    groupName="arcdemo" # Name of resource group for the connected cluster
    clusterName="${groupName}-cluster" # Name of the connected cluster resource
    workspaceName="$groupName-workspace" # Name of the Log Analytics workspace
    extensionName="appservice-ext" # Name of the App Service extension
    namespace="appservice-ns" # Namespace in your cluster to install the extension and provision resources
    kubeEnvironmentName="hybrid-app-service" # Name of the App Service Kubernetes environment resource
    customLocationName="traefik-Turkey" # Name of the custom location
    storageaccountname="functiontooling"
    logicAppName="HybridLogicApp"
    
# appservicek8senvironment

    az group create -g $aksClusterGroupName -l $resourceLocation
    az aks create --resource-group $aksClusterGroupName --name $aksName --enable-aad --generate-ssh-keys
    infra_rg=$(az aks show --resource-group $aksClusterGroupName --name $aksName --output tsv --query nodeResourceGroup)
    az network public-ip create --resource-group $infra_rg --name MyPublicIP --sku STANDARD
    staticIp=$(az network public-ip show --resource-group $infra_rg --name MyPublicIP --output tsv --query ipAddress)

#   create aks cluster
    
    az aks get-credentials --resource-group $aksClusterGroupName --name $aksName --admin

    kubectl get ns

    az group create -g $groupName -l $resourceLocation

    az connectedk8s connect --resource-group $groupName --name $clusterName

    az connectedk8s show --resource-group $groupName --name $clusterName

#   create az monitor workspace 
    az monitor log-analytics workspace create \
        --resource-group $groupName \
        --workspace-name $workspaceName

    logAnalyticsWorkspaceId=$(az monitor log-analytics workspace show \
        --resource-group $groupName \
        --workspace-name $workspaceName \
        --query customerId \
        --output tsv)
    
    logAnalyticsWorkspaceIdEnc=$(printf %s $logAnalyticsWorkspaceId | base64) # Needed for the next step
    
    logAnalyticsKey=$(az monitor log-analytics workspace get-shared-keys \
        --resource-group $groupName \
        --workspace-name $workspaceName \
        --query primarySharedKey \
        --output tsv)
    
    logAnalyticsKeyEncWithSpace=$(printf %s $logAnalyticsKey | base64)
    
    logAnalyticsKeyEnc=$(echo -n "${logAnalyticsKeyEncWithSpace//[[:space:]]/}") # Needed for the next step

    az k8s-extension create \
        --resource-group $groupName \
        --name $extensionName \
        --cluster-type connectedClusters \
        --cluster-name $clusterName \
        --extension-type 'Microsoft.Web.Appservice' \
        --release-train stable \
        --auto-upgrade-minor-version true \
        --scope cluster \
        --release-namespace $namespace \
        --configuration-settings "Microsoft.CustomLocation.ServiceAccount=default" \
        --configuration-settings "appsNamespace=${namespace}" \
        --configuration-settings "clusterName=${kubeEnvironmentName}" \
        --configuration-settings "loadBalancerIp=${staticIp}" \
        --configuration-settings "keda.enabled=true" \
        --configuration-settings "buildService.storageClassName=default" \
        --configuration-settings "buildService.storageAccessMode=ReadWriteOnce" \
        --configuration-settings "customConfigMap=${namespace}/kube-environment-config" \
        --configuration-settings "envoy.annotations.service.beta.kubernetes.io/azure-load-balancer-resource-group=${aksClusterGroupName}" \
        --configuration-settings "logProcessor.appLogs.destination=log-analytics" \
        --configuration-protected-settings "logProcessor.appLogs.logAnalyticsConfig.customerId=${logAnalyticsWorkspaceIdEnc}" \
        --configuration-protected-settings "logProcessor.appLogs.logAnalyticsConfig.sharedKey=${logAnalyticsKeyEnc}"

    extensionId=$(az k8s-extension show \
        --cluster-type connectedClusters \
        --cluster-name $clusterName \
        --resource-group $groupName \
        --name $extensionName \
        --query id \
        --output tsv)
    
    az resource wait --ids $extensionId --custom "properties.installState!='Pending'" --api-version "2020-07-01-preview"
    
    kubectl get pods -n $namespace
    
#   create custom location    

    connectedClusterId=$(az connectedk8s show --resource-group $groupName --name $clusterName --query id --output tsv)

    az customlocation create \
        --resource-group $groupName \
        --name $customLocationName \
        --host-resource-id $connectedClusterId \
        --namespace $namespace \
        --cluster-extension-ids $extensionId
    
    az customlocation show \
    --resource-group $groupName \
    --name $customLocationName
    
    customLocationId=$(az customlocation show \
    --resource-group $groupName \
    --name $customLocationName \
    --query id \
    --output tsv)
#   create app service     
    az appservice kube create \
    --resource-group $groupName \
    --name $kubeEnvironmentName \
    --custom-location $customLocationId \
    --static-ip $staticIp
    
    az appservice kube show \
    --resource-group $groupName \
    --name $kubeEnvironmentName

#   create web app

    az extension add --upgrade --yes --name customlocation
    az extension remove --name appservice-kube
    az extension add --yes --source "https://aka.ms/appsvc/appservice_kube-latest-py2.py3-none-any.whl"

    az webapp create \
        --resource-group $groupName \
        --name hybrid-app-node \
        --custom-location $customLocationId \
        --runtime 'NODE|12-lts'


    git clone https://github.com/Azure-Samples/nodejs-docs-hello-world
    cd nodejs-docs-hello-world
    zip -r package.zip .
    az webapp deployment source config-zip --resource-group $groupName --name hybrid-app-node --src package.zip

#   create function 

    func init LocalFunctionProj --dotnet
    cd LocalFunctionProj
    func new --name HttpExample --template "HTTP trigger" --authlevel "anonymous"
    func start

    connectedClusterId=$(az connectedk8s show --resource-group $groupName --name $clusterName --query id --output tsv)
    
    customLocationId=$(az customlocation show \
    --resource-group $groupName \
    --name $customLocationName \
    --query id \
    --output tsv)
    
    az storage account create --name $storageaccountname --location $resourceLocation --resource-group $groupName --sku Standard_LRS
    
    az functionapp create --resource-group $groupName --name hybrid-func-app --custom-location $customLocationId --storage-account $storageaccountname --functions-version 3 --runtime dotnet
    
    func azure functionapp publish hybrid-func-app
    
#   Create logic apps
    az extension add --yes --source "https://aka.ms/logicapp-latest-py2.py3-none-any.whl"
    az logicapp create --resource-group $groupName --name $logicAppName --storage-account $storageaccountname --custom-location $customLocationName
    
    #az logicapp create --resource-group $groupName --name $logicAppName --storage-account $storageaccountname --custom-location $customLocationName --deployment-container-image-name myacr.azurecr.io/myimage:tag --docker-registry-server-password passw0rd --docker-registry-server-user MyUser
    
    az logicapp deployment source config-zip --name $logicAppName --resource-group $groupName --src C:\uploads\v22.zip 
    az logicapp start --name $logicAppName --resource-group $groupName
   
    
