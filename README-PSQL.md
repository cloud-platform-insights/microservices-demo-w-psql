# instructions for adding PostgreSQL component 
(adapted from https://cloud.google.com/sql/docs/postgres/connect-instance-kubernetes)

## Get to zero
Do all the regular stuff in README.md -- make sure it works.

## Enable services

```sh
gcloud services enable compute.googleapis.com sqladmin.googleapis.com \
     container.googleapis.com artifactregistry.googleapis.com cloudbuild.googleapis.com
```

## Create Cloud SQL instance

```sh
gcloud compute addresses create google-managed-services-default \
--global \
--purpose=VPC_PEERING \
--prefix-length=16 \
--description="peering range for Google" \
--network=default
```

```sh
gcloud services vpc-peerings connect \
--service=servicenetworking.googleapis.com \
--ranges=google-managed-services-default \
--network=default
```

```sh
export DB_PASS=<make-a-password>
```

```sh
gcloud beta sql instances create online-boutique-db \
--database-version=POSTGRES_13 \
 --cpu=1 \
 --memory=4GB \
 --region=us-central \
 --root-password=$DB_PASS \
 --no-assign-ip \
--network=default
```

```sh
gcloud sql instances patch online-boutique-db --require-ssl
```

```sh
gcloud sql databases create products --instance=online-boutique-db
```

```sh
gcloud sql users create dbuser \
--instance=online-boutique-db \
--password=$DB_PASS
```

```sh
EXPORT CLUSTER_NAME=<name-of-cluster-where-onlineboutique-is-deployed>
```

```sh
gcloud iam service-accounts create gke-psql-service-account \
  --display-name="Online Boutique PSQL Service Account"
```

```sh
EXPORT PROJECT_ID=<your-project-id>
```

```sh
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:gke-quickstart-service-account@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/cloudsql.client"
```

```sh
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:gke-quickstart-service-account@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/logging.logWriter"
```

```sh
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:gke-quickstart-service-account@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.reader"
```

## Add connectivity between k8s and cloud sql
TODO: just write this file into the repo

Make a manifest for the service account; name it `sqlserviceaccount.yaml` and save it in the `kubernetes-manifests` folder:
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ksa-cloud-sql
```

Apply it:
```
kubectl apply -f kubernetes-manifests/sqlserviceaccount.yaml
```

Bind the k8s service account to GCP service account:

```sh
gcloud iam service-accounts add-iam-policy-binding \
  --role="roles/iam.workloadIdentityUser" \
  --member="serviceAccount:cpet-stanke-sandbox.svc.id.goog[default/ksa-cloud-sql]" \
  gke-quickstart-service-account@cpet-stanke-sandbox.iam.gserviceaccount.com
```

```sh
kubectl annotate serviceaccount \
  ksa-cloud-sql  \
  iam.gke.io/gcp-service-account=gke-psql-service-account@$PROJECT_ID$.iam.gserviceaccount.com
```

```sh
kubectl create secret generic gke-cloud-sql-secrets \
  --from-literal=database=products \
  --from-literal=username=dbuser \
  --from-literal=password=$DB_PASS
```

## TODO: Update productcatalogservice to actually use the db