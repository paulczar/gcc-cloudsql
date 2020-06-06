# Example of deploying a CloudSQL instance with Google Config Connector

## Prerequisites

We'll be using the `k14s` tools `ytt` and `kapp` to do some light templating of the kube manifests.

1. Install k14s via curl|bash:

```bash
curl -s -L https://k14s.io/install.sh | \
  K14SIO_INSTALL_BIN_DIR=~/bin bash
```

2. Install gcloud SDK CLI via [these instructions](https://cloud.google.com/sdk/docs/quickstarts).


## Prepare Local Environment

1. Clone this repository:

```bash
git clone https://github.com/paulczar/gcc-cloudsql
cd gcc-cloudsql
```


## Prepare Google Cloud Account

1. Ensure that you have enabled the [Cloud IAM Service Account Credentials API](https://console.cloud.google.com/apis/api/iamcredentials.googleapis.com/overview?_ga=2.231642838.851680512.1591144110-657035042.1588876317), the Service Networking API, and the Service Management API:

```bash
gcloud services enable \
  servicenetworking.googleapis.com \
  servicemanagement.googleapis.com \
  iamcredentials.googleapis.com
```

## Create GKE Cluster

1. Export your GCP Project ID to an environment variable:

```bash
PROJECT_ID=$(gcloud config get-value project)
```

2. Create a Small GKE Cluster using `--workload-pool` to enable [workload identity](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity) and `--enable-ip-alias` to create a [VPC-native cluster](https://cloud.google.com/kubernetes-engine/docs/how-to/alias-ips):

```bash
gcloud container clusters create gcc-cloudsql \
  --num-nodes=1 --zone us-central1-c \
  --cluster-version 1.16 --machine-type n1-standard-2 \
  --workload-pool=${PROJECT_ID}.svc.id.goog \
  --enable-ip-alias
```

*If you want to use a different network you can, just make sure you deploy the GKE cluster to the same network as the SQL instance Peer.*

3. Once the cluster is created check your access:

```bash
kubectl cluster-info
```

## Create Private IP Peering

**This only needs to be done once per network per project**

*You could do this with GCC, however GCC won't wait for the Peer to be ready before trying to start the SQL Instance resulting in errors. For an example of doing this via GCC see `./ytt/infrastructure`.*

1. Create a VPC Peering range:

```bash
gcloud compute addresses create cloudsql-peer \
    --global \
    --purpose=VPC_PEERING \
    --prefix-length=16 \
    --description="peering range for CloudSQL" \
    --network=default \
    --project=$PROJECT_ID
```

2. Peer that range with our default network:

```bash
gcloud services vpc-peerings connect \
    --service=servicenetworking.googleapis.com \
    --ranges=cloudsql-peer \
    --network=default \
    --project=$PROJECT_ID
```

*If this command fails with `Cannot modify allocated ranges in CreateConnection` then rerun the command but replace `connect` with `update --force`.*

## Install Google Config Connector

*I have included the manifests for [installing GCC](https://cloud.google.com/config-connector/docs/how-to/install-upgrade-uninstall) in workload-identity mode in this repository for ease of use.*

1. Create a service account for GCC:

```bash
gcloud iam service-accounts create cnrm-system
```

2. Bind the `roles/owner` to the service account:

```bash
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:cnrm-system@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/owner"
```

3. Bind `roles/iam.workloadIdentityUser` to the `cnrm-controller-manager` Kubernetes Service Account in the `cnrm-system` Namespace:

```bash
gcloud iam service-accounts add-iam-policy-binding \
  --role="roles/iam.workloadIdentityUser" \
  --member="serviceAccount:${PROJECT_ID}.svc.id.goog[cnrm-system/cnrm-controller-manager]" \
cnrm-system@${PROJECT_ID}.iam.gserviceaccount.com
```

4. Create a Kubernetes manifest for GCC using `ytt`:

```bash
ytt -f ./ytt/gcc \
  --data-value "gcp.projectID=$PROJECT_ID" \
  > ./manifests/gcc.yaml
```

5. Deploy GCC using `kapp`:

```bash
kapp deploy -a gcc -y \
  -f ./manifests/gcc.yaml
```

*Note: You could use `kubectl apply -f ./manifests/gcc.yaml` for the above, but `kapp` gives you a better view of what is going on.*

6. Wait until GCC is running:

```bash
$ kubectl wait -n cnrm-system --for=condition=Ready pod --all
pod/cnrm-controller-manager-0 condition met
pod/cnrm-deletiondefender-0 condition met
pod/cnrm-resource-stats-recorder-88b54bdd7-6hq9p condition met
pod/cnrm-webhook-manager-7b4db8b7d5-5llfs condition met
```

## Deploy your CloudSQL Instance

1. Create a Kubernetes manifest for CloudSQL using `ytt`:

```bash
ytt -f ./ytt/cloudsql \
  --data-value "gcp.projectID=$PROJECT_ID" \
  --data-value "db.rootPassword=this-is-a-bad-password" \
  --data-value "name=example" \
  --data-value "namespace=cloudsql" \
  > ./manifests/cloudsql.yaml
```

2. Deploy CloudSQL using `kapp`:

```bash
kapp deploy -a cloudsql -y \
  -f ./manifests/cloudsql.yaml
```

3. Wait until the database is ready:

*It shouldn't take more than five minutes, but if it does ... just wait longer.*

```bash
kubectl wait -n cloudsql --for=condition=Ready \
  sqlinstance/example-db --timeout=300s
```

## Verify you can connect

1. Get the IP address of the database:

```bash
gcloud sql instances describe example-db --format json | jq -r '.ipAddresses[0].ipAddress'
IP=$(!!)
```

2. Remind yourself of the password:

```bash
kubectl -n cloudsql get secret example-db -o json | jq -r .data.password | base64 --decode
```

2. Get a prompt in a Pod with the `psql` client:

```bash
kubectl -n cloudsql run --env IP=$IP -ti --restart=Never --image postgres:13-alpine --rm psql -- sh
```

3. Connect to your database from inside that pod:

```bash
$ psql -h $IP --username=example -d postgres
Password for user example: *******
psql (13beta1, server 9.6.16)
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

postgres=>

```

*Press Ctrl-D to exit psql*

4. Connect to your database via the sql proxy:

```bash
$ psql -h example-db-proxy --username=example -d postgres
Password for user example: *****
psql (13beta1, server 9.6.16)
Type "help" for help.

postgres=> 

```

## Cleanup

Once you're finished, you can clean up like so:

1. Delete the cloudsql resources:

```bash
kapp delete -y -a cloudsql
```

2. Delete the GKE Cluster

```bash
gcloud container clusters delete gcc-cloudsql
```

3. Delete the rest of the resources