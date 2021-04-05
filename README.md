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
az aks nodepool add --resource-group $RGNAME \
--cluster-name $CLUSTERNAME \
--name agonesnp2 \
--node-vm-size Standard_A4_v2 \
--node-count 5 \
--mode User \
--node-taints=agones.dev/agones-system=true:NoExecute \
--labels=agones.dev/agones-system=true \
--enable-node-public-ip \
--kubernetes-version 1.18.14

az aks nodepool list --resource-group $RGNAME --cluster-name $CLUSTERNAME
az aks nodepool delete --resource-group $RGNAME --cluster-name $CLUSTERNAME -n agonesnp1

# agones manual install
helm repo add agones https://agones.dev/chart/stable
helm repo update

kubectl create ns agones-system

helm install agones-release --namespace agones-system --create-namespace agones/agones

kubectl create -f ./arc/manifests/gameserver.yaml
kubectl create -f ./arc/manifests/fleet.yaml

kubectl run ubuntu --rm -it --image=ubuntu -- bash
apt update
apt install netcat 
nc -u 10.240.0.9 7637
nc -u 13.84.205.169 7229

az vmss list-instance-public-ips -g MC_agones_briar-agones_southcentralus -n aks-agonesnp2-21002717-vmss

# arc Setup
az connectedk8s list -g $RGNAME
az connectedk8s connect --name $CLUSTERNAME --resource-group $RGNAME -l $LOCATION

az k8s-configuration create \
--name agones-base \
--cluster-name $CLUSTERNAME \
--resource-group $RGNAME \
--operator-instance-name agones-helm \
--operator-namespace flux \
--operator-params='--git-readonly --git-path=arc/agones-helm --git-branch main --sync-garbage-collection' \
--enable-helm-operator \
--helm-operator-chart-version='1.2.0' \
--helm-operator-params='--set helm.versions=v3' \
--repository-url https://github.com/chzbrgr71/agones-aks.git \
--scope cluster \
--cluster-type connectedClusters

az k8s-configuration list --cluster-name $CLUSTERNAME --resource-group $RGNAME --cluster-type connectedClusters

az k8s-configuration show --name agones-base --cluster-name $CLUSTERNAME --resource-group $RGNAME --cluster-type connectedClusters -o json

az k8s-configuration delete --name agones-base --cluster-name $CLUSTERNAME --resource-group $RGNAME --cluster-type connectedClusters


```