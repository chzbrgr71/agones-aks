## Experimental - Agones on AKS

#### Setup Notes

Links:
* https://agones.dev/site/docs/installation/creating-cluster/aks
* https://agones.dev/site/docs/installation/install-agones/helm 

```bash

# create AKS cluster
export CLUSTERNAME=briar-agones
export K8SVERSION=1.18.14
export VMSIZE=Standard_D2_v2
export NODECOUNT=3
export RGNAME=agones
export LOCATION=southcentralus
export AZUREMONITOR=/subscriptions/471d33fd-a776-405b-947c-467c291dc741/resourcegroups/monitoring/providers/microsoft.operationalinsights/workspaces/briar-aks-monitoring
export SECOND_NODEPOOL=agonesnp1

az group create --name $RGNAME --location $LOCATION

az aks create \
    --resource-group $RGNAME \
    --name $CLUSTERNAME \
    --node-count $NODECOUNT \
    --kubernetes-version $K8SVERSION \
    --location $LOCATION \
    --vm-set-type VirtualMachineScaleSets \
    --enable-managed-identity \
    --workspace-resource-id $AZUREMONITOR \
    --enable-addons monitoring \
    --no-wait

az aks get-credentials -n $CLUSTERNAME -g $RGNAME

# add NSG
export INFRA_RG=$(az aks show -g $RGNAME -n $CLUSTERNAME -o tsv --query nodeResourceGroup)
export NSG_NAME=aks-agentpool-21002717-nsg
export VMSS_NAME=aks-nodepool1-21002717-vmss

az network nsg rule create \
  --resource-group $INFRA_RG \
  --nsg-name $NSG_NAME \
  --name AgonesUDP \
  --access Allow \
  --protocol Udp \
  --direction Inbound \
  --priority 520 \
  --source-port-range "*" \
  --destination-port-range 7000-8000

# add agones nodepool with PIP per node
az aks nodepool add \
    --resource-group $RGNAME \
    --cluster-name $CLUSTERNAME \
    --name $SECOND_NODEPOOL \
    --node-count 5 \
    --node-vm-size $VMSIZE \
    --mode user \
    --labels app=agones \
    --enable-node-public-ip \
    --no-wait

az aks nodepool list --resource-group $RGNAME --cluster-name $CLUSTERNAME

# agones manual install

helm repo add agones https://agones.dev/chart/stable
helm repo update

kubectl create ns agones-system

helm install agones-release agones/agones \
    --namespace agones-system \
    --create-namespace \
    --set 'agones.controller.nodeSelector.app=agones' \
    --set 'agones.ping.nodeSelector.app=agones' \
    --set 'agones.allocator.nodeSelector.app=agones'

kubectl create -f ./arc/manifests/gameserver.yaml
kubectl create -f ./arc/manifests/fleet.yaml

kubectl run ubuntu --rm -it --image=ubuntu -- bash
apt update
apt install netcat 
nc -u 10.240.0.11 7893

# Arc Setup
az connectedk8s list -g $RGNAME
az connectedk8s connect --name $CLUSTERNAME --resource-group $RGNAME -l $LOCATION

az k8s-configuration create \
--name agones-base2 \
--cluster-name $CLUSTERNAME \
--resource-group $RGNAME \
--operator-instance-name agones-helm2 \
--operator-namespace flux2 \
--operator-params='--git-readonly --git-path=arc/agones-helm --git-branch main --sync-garbage-collection' \
--enable-helm-operator \
--helm-operator-chart-version='1.2.0' \
--helm-operator-params='--set helm.versions=v3' \
--repository-url https://github.com/chzbrgr71/agones-aks.git \
--scope namespace \
--cluster-type connectedClusters

az k8s-configuration list --cluster-name $CLUSTERNAME --resource-group $RGNAME --cluster-type connectedClusters

az k8s-configuration show --name agones-base --cluster-name $CLUSTERNAME --resource-group $RGNAME --cluster-type connectedClusters -o json

az k8s-configuration delete --name agones-base --cluster-name $CLUSTERNAME --resource-group $RGNAME --cluster-type connectedClusters

az k8s-configuration create --name azure-arc-sample --cluster-name $CLUSTERNAME --resource-group $RGNAME --operator-instance-name flux --operator-namespace arc-k8s-demo --operator-params='--git-readonly --git-path=releases' --enable-helm-operator --helm-operator-chart-version='1.2.0' --helm-operator-params='--set helm.versions=v3' --repository-url https://github.com/Azure/arc-helm-demo.git --scope namespace --cluster-type connectedClusters

az k8s-configuration show --name azure-arc-sample --cluster-name $CLUSTERNAME --resource-group $RGNAME --cluster-type connectedClusters -o json

```