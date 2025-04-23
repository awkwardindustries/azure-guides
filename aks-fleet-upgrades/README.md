# AKS Fleet Manager Upgrades

## Cluster Setup

```bash
# Setup environment
RG=eph-rg-fleet-upgrade-demo
LOC=eastus2
FLEET=fleetdemo
CLUSTERS_LIST="dev-canary dev test-canary test stg-canary stg prd"
UPDATE_STRATEGY_NAME=staged-update-strategy-cli
AUTOUPGRADE_PROFILE_NAME=autoupgrade-profile-name-cli

# Create the resource group
az group create -g $RG -l $LOC

# Create some AKS clusters
for cluster_name in $CLUSTERS_LIST
do
  az aks create -g $RG -n $cluster_name -c 1 --no-wait
done

# Get credentials to access clusters
do
  az aks get-credentials -g $RG -n $cluster_name
done
```

## Fleet Setup

```bash
az extension add --name fleet
az extension update --name fleet

# Create the fleet
az fleet create --resource-group $RG --name ${FLEET} --location $LOC

# Add members (clusters) to fleet
for cluster_name in $CLUSTERS_LIST
do
  az fleet member create \
  --resource-group $RG \
  --fleet-name $FLEET \
  --name $cluster_name \
  --member-cluster-id $(az aks show -g $RG -n $cluster_name --query id -o tsv) \
  --no-wait
done
```

## Upgrade Setup

### Current versions and target versions
```bash
# See what versions clusters are running and what are available

az aks list -g $RG -o table
az aks get-versions --location $LOC --output table

# Version to upgrade clusters
TARGET_K8S_VERSION="1.32.1"
```

### Assign clusters to Update Groups

```bash
az fleet member update \
--resource-group $RG \
--fleet-name $FLEET \
--name dev-canary \
--update-group dev-canary

az fleet member update \
--resource-group $RG \
--fleet-name $FLEET \
--name dev \
--update-group dev

az fleet member update \
--resource-group $RG \
--fleet-name $FLEET \
--name tst-canary \
--update-group test

az fleet member update \
--resource-group $RG \
--fleet-name $FLEET \
--name tst \
--update-group test

az fleet member update \
--resource-group $RG \
--fleet-name $FLEET \
--name stg-canary \
--update-group stage

az fleet member update \
--resource-group $RG \
--fleet-name $FLEET \
--name stg \
--update-group stage

az fleet member update \
--resource-group $RG \
--fleet-name $FLEET \
--name prod \
--update-group prod
```

### Create an Update Strategy
```bash
# Create a stages JSON file
cat <<EOF > stages.json
{
    "stages": [
        {
            "name": "canary-group",
            "groups": [
                {
                    "name": "dev-canary"
                },
                {
                    "name": "tst-canary"
                },
                {
                    "name": "stg-canary"
                }
            ],
            "afterStageWaitInSeconds": 30
        },
        {
            "name": "dev-group",
            "groups": [
                {
                    "name": "dev"
                }
            ],
            "afterStageWaitInSeconds": 30
        },
        {
            "name": "test-group",
            "groups": [
                {
                    "name": "tst"
                }
            ],
            "afterStageWaitInSeconds": 30
        },
        {
            "name": "stage-group",
            "groups": [
                {
                    "name": "stg"
                }
            ],
            "afterStageWaitInSeconds": 30
        },
        {
            "name": "prod-group",
            "groups": [
                {
                    "name": "prod"
                }
            ]
        }
    ]
}
EOF

# Create the update strategy
az fleet updatestrategy create \
--resource-group $RG \
--fleet-name $FLEET \
--name $UPDATE_STRATEGY_NAME \
--stages stages.json

# Grab the resource ID for the update strategy
STRATEGY_ID=$(az fleet updatestrategy show -g $RG -f $FLEET -n $UPDATE_STRATEGY_NAME --query id -o tsv)
```

### Option: Create an Auto-Upgrade Profile using the Update Strategy
```bash
az fleet autoupgradeprofile create \
--resource-group $RG \
--fleet-name $FLEET \
--name $AUTOUPGRADE_PROFILE_NAME \
--channel Stable \
--update-strategy-id $STRATEGY_ID
```

## Run an Upgrade

> **IMPORTANT**: If the cluster members have *planned maintenance windows* they will be honored. Your runs may be `Pending` until the planned maintenance window opens.

### Option: Manually trigger an Auto-Upgrade Profile run (versus waiting for a channel trigger)
```bash
az fleet autoupgradeprofile generate-update-run \
--resource-group $RG \
--fleet-name $FLEET \
--name $AUTOUPGRADE_PROFILE_NAME
```

### Option: Manually trigger a run from an Update Strategy
```bash
az fleet updaterun create \
--resource-group $RG \
--fleet-name $FLEET \
--name test-run-manual-from-strategy-cli \
--upgrade-type Full \
--kubernetes-version TARGET_K8S_VERSION \
--node-image-selection Latest \
--update-strategy-name $UPDATE_STRATEGY_NAME
```

### Option: Manually trigger a run from a stages specification file
```bash
az fleet updaterun create \
--resource-group $RG \
--fleet-name $FLEET \
--name test-run-manual-from-stages \
--upgrade-type Full \
--kubernetes-version TARGET_K8S_VERSION \
--node-image-selection Latest \
--stages stages.json
```