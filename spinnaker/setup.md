# Spinnaker setup

ToDo:
- 酔っ払っていつもの癖で英語で書いてしまったのであとで日本語版も書く。
-

## Prerequisites

- Running kubernetes clusters
  - We assume that there are two clusters, for staging and for production, on GKE.
  - Kubernetes version: 1.10+

## Reference

Set up Spinnaker: https://www.spinnaker.io/setup/


## Setup Spinnaker

### Install Halyard

Halyard manages the lifecycle of Spinnaker deployment. First of all, you'll need to have halyard on your working environment.

Ubuntu:
```
curl -O https://raw.githubusercontent.com/spinnaker/halyard/master/install/debian/InstallHalyard.sh
```
```
sudo bash InstallHalyard.sh
```

### Choose Cloud Providers

Spinnaker can be deployed on various IaaS platforms.
https://www.spinnaker.io/setup/install/providers/

This time we use `Kubernetes Provider V2 (Manifest Based)`.


#### Get kubernetes credentials

We assume that there are two k8s clusters:
- `showks-cluster-stg` : Deployment target for staging, and spinnaker itself
- `showks-cluster-prod` : Deployment target for production

Halyard and Spinnaker need to have credentials for both. If you have credentials as "cluster-admin" of those clusters, the easiest way is to give those to Halyard.
If you are running clusters on GKE, you can get credentials like this:
```
gcloud config set project [PROJECT_ID]
gcloud config set compute/zone [COMPUTE_ZONE]

CLUSTER_PROD=showks-cluster-prod
CLUSTER_STG=showks-cluster-stg

gcloud container clusters get-credentials $CLUSTER_PROD
gcloud container clusters get-credentials $CLUSTER_STG

```

Then you can add accounts to Halyard from kubectl contexts.

```
CONTEXT_PROD=`kubectl config get-contexts | grep $CLUSTER_PROD | awk '{print $2}'`
CONTEXT_STG=`kubectl config get-contexts | grep $CLUSTER_STG | awk '{print $2}'`

hal config provider kubernetes enable
hal config provider kubernetes account add ${CLUSTER_PROD}-account --provider-version v2 --context $CONTEXT_PROD
hal config provider kubernetes account add ${CLUSTER_STG}-account --provider-version v2 --context $CONTEXT_STG
hal config features edit --artifacts true
```

### Storage backend

Spinnaker requires persistent storage as backend. https://www.spinnaker.io/setup/install/storage/
This time we use Google Cloud Storage. If you are using on-prem k8s and want to use Minio, refer *TBD*

```
SERVICE_ACCOUNT_NAME=spinnaker-gcs-account
SERVICE_ACCOUNT_DEST=~/.gcp/gcs-account.json
PROJECT=$(gcloud info --format='value(config.project)')
BUCKET_LOCATION=asia-northeast1

gcloud iam service-accounts create $SERVICE_ACCOUNT_NAME --display-name $SERVICE_ACCOUNT_NAME

SA_EMAIL=$(gcloud iam service-accounts list \
  --filter="displayName:$SERVICE_ACCOUNT_NAME" \
  --format='value(email)')

gcloud projects add-iam-policy-binding $PROJECT --role roles/storage.admin --member serviceAccount:$SA_EMAIL

mkdir -p $(dirname $SERVICE_ACCOUNT_DEST)
gcloud iam service-accounts keys create $SERVICE_ACCOUNT_DEST --iam-account $SA_EMAIL

hal config storage gcs edit --project $PROJECT --bucket-location $BUCKET_LOCATION --json-path $SERVICE_ACCOUNT_DEST
hal config storage edit --type gcs
```


### Deploy spinnaker

This time we deploy Spinnaker on top of STG cluster.

```
hal version list
hal config version edit --version 1.10.1
hal config deploy edit --type distributed --account-name ${CLUSTER_STG}-account

hal deploy apply
```

## Expose Spinnaker to the Internet

By default, Spinnaker UI and API can be accessed through SSL tunnel. This time we expose it to the internet, since
- Administrators are on the internet.
- showKs uses GitHub as source code repository, so webhooks have to be reachable to the API.
