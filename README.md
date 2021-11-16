# Urbit Ops

Made this repo to share some of what I use to deploy Urbit to Kubernetes running on Linode, with some DNS, Cloud Builds and Container Registry running in Google cloud. 

I used to run everything in Google cloud, but have been migrating out because of cost. For example, a working private docker registry is included as part of the linode kubernetes, though I'm not actually using it here.

These files are incomplete, as I pulled them out of a large private repo that contains additional operation files. These files will not necessarily be updated and are for informational purposes only.

# Requirements

 - Helm3
 - Terraform 0.15+
 - kubectl
 - gcloud
 - sops
 - docker

# Terraform

Apply terraform in this order:

 - cloud/bootstrap
 - cloud/gcp

Run `bootstrap.sh` in cloud/linode to create cluster. This gets you as far as a K8s cluster running on Linode, with DNS on GCP.

# Containers

containers directory contains various container builds. These containers are pushed to GCR.

# Berglas

Secrets (master keys, mainly) are stored in berglas. This is likely overkill for a single urbit keyfile, but I already have all of the machinery in place. Regular Kubernetes secrets can be used instead.

## Helper variables

Before running any commands from this README, first set the project id

```bash
export PROJECT_ID=$(gcloud config get-value project)
```

## Load cluster with GCP service account

This steps and this secret is intionally left out of the repo.

In order to decode secrets in berglas vault, the cluster needs to be able to decrypt and access storage. 

Log in to gcloud and set the default application credentials:

```bash
gcloud auth application-default login
```

Enable some Google APIs:
```bash
gcloud services enable --project ${PROJECT_ID} \
  cloudkms.googleapis.com \
  storage-api.googleapis.com \
  storage-component.googleapis.com
```

Create a new service account, download the service account key, create the secret, delete the key:

```bash
gcloud iam service-accounts create vault-decrypt --display-name "Vault decrypter" --description "Workload service account to decrypt secrets from vault"
gcloud iam service-accounts keys create key.json --iam-account vault-decrypt@$PROJECT_ID.iam.gserviceaccount.com
JSON_KEY=$(cat key.json | base64 -w 0) envsubst '$$JSON_KEY' < ops/bootstrap/manifests/secrets.yaml | kubectl apply --namespace soapbubble -f -
rm key.json
```

## berglas

Berglas is used to manage secrets.

When working with berglas, set the BUCKET_IT var

```bash
export BUCKET_ID=${PROJECT_ID}-secrets
```

### Initialization

Only needs to be done once per GCP project

#### Bootstrap bucket and KMS

```bash
berglas bootstrap -b $BUCKET_ID -l us-central1 -p $PROJECT_ID
```

#### Install the berglas mutatingwebhook

Make sure kubectl is pointing at the k8s cluster you want to install to

```bash
helm upgrade --install --create-namespace --values /kubernetes/live/linode/berglas-mutatingwebhook/values.yaml berglas kubernetes/charts/internal/berglas-mutatingwebhook/0.0.1
```

Create or update the kubeconfig secret

```bash
berglas update $BUCKET_ID/sampel-palnet @.sampel-palnet.key --key projects/$PROJECT_ID/locations/global/keyRings/berglas/cryptoKeys/berglas-key --create-if-missing
```

If needed, grant cloudbuild access to this secret:

```bash
PROJECT_NUMBER=$(gcloud projects describe ${PROJECT_ID} --format 'value(projectNumber)') SA_EMAIL=${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com berglas grant ${BUCKET_ID}/kubeconfig --member serviceAccount:${SA_EMAIL}
```
