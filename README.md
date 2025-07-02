This repo contains a CPET-specific fork of online-microservices-demo, in which the product catalog is provided by a SQL database, rather than a JSON file. Having a dependency on a traditional database makes it more representative of a legacy enterprise workload.

Some elements of online-microservices-demo (esp. alternative installs etc) have been stripped out.

This README file refers only to this fork. The original README has been renamed to README-UPSTREAM.

---

### Infra
1. Create a Postgres db server with a public IP, accessible from `0.0.0.0/0` [TODO: make this private and access via cloudsqlproxy]
2. Create a database called `products`
3. Run the following SQL statements:

    ```sql
    CREATE TABLE catalog_items (id TEXT PRIMARY KEY, name TEXT, description TEXT, picture TEXT, price_usd_currency_code TEXT, price_usd_units INTEGER, price_usd_nanos BIGINT, categories TEXT);
    INSERT INTO catalog_items (id, name, description, picture, price_usd_currency_code, price_usd_units, price_usd_nanos, categories) VALUES ('OLJCESPC7Z', 'Sunglasses', 'Add a modern touch to your outfits with these sleek aviator sunglasses.', '/static/img/products/sunglasses.jpg', 'USD', 19, 990000000, 'accessories');
    INSERT INTO catalog_items (id, name, description, picture, price_usd_currency_code, price_usd_units, price_usd_nanos, categories) VALUES ('66VCHSJNUP', 'Tank Top', 'Perfectly cropped cotton tank, with a scooped neckline.', '/static/img/products/tank-top.jpg', 'USD', 18, 990000000, 'clothing,tops');
    INSERT INTO catalog_items (id, name, description, picture, price_usd_currency_code, price_usd_units, price_usd_nanos, categories) VALUES ('1YMWWN1N4O', 'Watch', 'This gold-tone stainless steel watch will work with most of your outfits.', '/static/img/products/watch.jpg', 'USD', 109, 990000000, 'accessories');
    INSERT INTO catalog_items (id, name, description, picture, price_usd_currency_code, price_usd_units, price_usd_nanos, categories) VALUES ('L9ECAV7KIM', 'Loafers', 'A neat addition to your summer wardrobe.', '/static/img/products/loafers.jpg', 'USD', 89, 990000000, 'footwear');
    INSERT INTO catalog_items (id, name, description, picture, price_usd_currency_code, price_usd_units, price_usd_nanos, categories) VALUES ('2ZYFJ3GM2N', 'Hairdryer', 'This lightweight hairdryer has 3 heat and speed settings. Its perfect for travel.', '/static/img/products/hairdryer.jpg', 'USD', 24, 990000000, 'hair,beauty');
    INSERT INTO catalog_items (id, name, description, picture, price_usd_currency_code, price_usd_units, price_usd_nanos, categories) VALUES ('0PUK6V6EV0', 'Candle Holder', 'This small but intricate candle holder is an excellent gift.', '/static/img/products/candle-holder.jpg', 'USD', 18, 990000000, 'decor,home');
    INSERT INTO catalog_items (id, name, description, picture, price_usd_currency_code, price_usd_units, price_usd_nanos, categories) VALUES ('LS4PSXUNUM', 'Salt & Pepper Shakers', 'Add some flavor to your kitchen.', '/static/img/products/salt-and-pepper-shakers.jpg', 'USD', 18, 490000000, 'kitchen');
    INSERT INTO catalog_items (id, name, description, picture, price_usd_currency_code, price_usd_units, price_usd_nanos, categories) VALUES ('9SIQT8TOJO', 'Bamboo Glass Jar', 'This bamboo glass jar can hold 57 oz (1.7 l) and is perfect for any kitchen.', '/static/img/products/bamboo-glass-jar.jpg', 'USD', 5, 490000000, 'kitchen');
    INSERT INTO catalog_items (id, name, description, picture, price_usd_currency_code, price_usd_units, price_usd_nanos, categories) VALUES ('6E92ZMYYFZ', 'Mug', 'A simple mug with a mustard interior.', '/static/img/products/mug.jpg', 'USD', 8, 990000000, 'kitchen');
    ```
4. Create a Postgres user and grant read permissions on `catalog_items` to it. Note the username and password for later.


### deploy locally for iterative development

1. Start a local k8s server, like minikube
2. `skaffold dev`
3. (open a new terminal for the following commands...)
4. Populate the secrets needed for db access (replace placeholders with actual values):
    ```sh
    kubectl create secret generic database-secrets \
      --from-literal=host=<DATABASE_HOST> \
      --from-literal=port=5432 \
      --from-literal=database=products \
      --from-literal=username=<DATABASE_USER> \
      --from-literal=password=<DATABASE_PASSWORD>
    ```
5. `kubectl port-forward deployment/frontend 8080:8080` to forward a port to the frontend service.
6. Navigate to `localhost:8080` to access the web frontend.


### deploy to a production[-like] environment
1. (if necessary) provision a kubernetes cluster
2. configure an artifact repo, authenticate for pushing to it, and grant pull access from the repository to the cluster
3. configure `kubectl` to access the cluster
4. Populate the secrets needed for db access (replace placeholders with actual values):
    ```sh
    kubectl create secret generic database-secrets \
      --from-literal=host=<DATABASE_HOST> \
      --from-literal=port=5432 \
      --from-literal=database=products \
      --from-literal=username=<DATABASE_USER> \
      --from-literal=password=<DATABASE_PASSWORD>
    ```
5. `skaffold config set default-repo <ARTIFACT_REPO_URL>`
6. `skaffold run -p gcp` [Note: I was unable to get this to work without the `-p gcb` config flag. IDK why.]
7. `kubectl get service frontend-external | awk '{print $4}'`
8. Visit `http://EXTERNAL_IP` in a web browser to access your instance of Online Boutique.




## OLDER stuff below (some of which is still relevant)





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
export CONNECTION_NAME=$(gcloud sql instances describe online-boutique-db --format='value(connectionName)')
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

Run the following SQL statements:

```sql
CREATE TABLE catalog_items (id TEXT PRIMARY KEY, name TEXT, description TEXT, picture TEXT, price_usd_currency_code TEXT, price_usd_units INTEGER, price_usd_nanos BIGINT, categories TEXT);
INSERT INTO catalog_items (id, name, description, picture, price_usd_currency_code, price_usd_units, price_usd_nanos, categories) VALUES ('OLJCESPC7Z', 'Sunglasses', 'Add a modern touch to your outfits with these sleek aviator sunglasses.', '/static/img/products/sunglasses.jpg', 'USD', 19, 990000000, 'accessories');
INSERT INTO catalog_items (id, name, description, picture, price_usd_currency_code, price_usd_units, price_usd_nanos, categories) VALUES ('66VCHSJNUP', 'Tank Top', 'Perfectly cropped cotton tank, with a scooped neckline.', '/static/img/products/tank-top.jpg', 'USD', 18, 990000000, 'clothing,tops');
INSERT INTO catalog_items (id, name, description, picture, price_usd_currency_code, price_usd_units, price_usd_nanos, categories) VALUES ('1YMWWN1N4O', 'Watch', 'This gold-tone stainless steel watch will work with most of your outfits.', '/static/img/products/watch.jpg', 'USD', 109, 990000000, 'accessories');
INSERT INTO catalog_items (id, name, description, picture, price_usd_currency_code, price_usd_units, price_usd_nanos, categories) VALUES ('L9ECAV7KIM', 'Loafers', 'A neat addition to your summer wardrobe.', '/static/img/products/loafers.jpg', 'USD', 89, 990000000, 'footwear');
INSERT INTO catalog_items (id, name, description, picture, price_usd_currency_code, price_usd_units, price_usd_nanos, categories) VALUES ('2ZYFJ3GM2N', 'Hairdryer', 'This lightweight hairdryer has 3 heat and speed settings. Its perfect for travel.', '/static/img/products/hairdryer.jpg', 'USD', 24, 990000000, 'hair,beauty');
INSERT INTO catalog_items (id, name, description, picture, price_usd_currency_code, price_usd_units, price_usd_nanos, categories) VALUES ('0PUK6V6EV0', 'Candle Holder', 'This small but intricate candle holder is an excellent gift.', '/static/img/products/candle-holder.jpg', 'USD', 18, 990000000, 'decor,home');
INSERT INTO catalog_items (id, name, description, picture, price_usd_currency_code, price_usd_units, price_usd_nanos, categories) VALUES ('LS4PSXUNUM', 'Salt & Pepper Shakers', 'Add some flavor to your kitchen.', '/static/img/products/salt-and-pepper-shakers.jpg', 'USD', 18, 490000000, 'kitchen');
INSERT INTO catalog_items (id, name, description, picture, price_usd_currency_code, price_usd_units, price_usd_nanos, categories) VALUES ('9SIQT8TOJO', 'Bamboo Glass Jar', 'This bamboo glass jar can hold 57 oz (1.7 l) and is perfect for any kitchen.', '/static/img/products/bamboo-glass-jar.jpg', 'USD', 5, 490000000, 'kitchen');
INSERT INTO catalog_items (id, name, description, picture, price_usd_currency_code, price_usd_units, price_usd_nanos, categories) VALUES ('6E92ZMYYFZ', 'Mug', 'A simple mug with a mustard interior.', '/static/img/products/mug.jpg', 'USD', 8, 990000000, 'kitchen');
```
