## Experimental - Agones on AKS

#### Setup Notes

```bash

# create AKS cluster
export CLUSTERNAME=agones-briar
export K8SVERSION=1.18.14
export VMSIZE=Standard_D2_v2
export NODECOUNT=5
export RGNAME=agones
export LOCATION=centralus
export AZUREMONITOR=/subscriptions/471d33fd-a776-405b-947c-467c291dc741/resourcegroups/monitoring/providers/microsoft.operationalinsights/workspaces/briar-aks-monitoring

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


# agones

```