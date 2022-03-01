# Welcome to this quick GCP DevOps Toolchain Demo! 
<img src="https://github.com/hamza-labs/gcp-devops-demo/blob/main/src/main/resources/static/resources/images/devsecops.png?raw=true" width="650"> 

### Step 1: Code and Clone your Repo! 

- Clone the DevOps Repo (Google Cloud Repository)
```
Open your Cloud Shell Editor shell.cloud.google.com 
New Terminal 
cd && mkdir projects && cd projects
gcloud source repos clone gcp-devops-demo --project=YOUR_PROJECT_ID
cd gcp-devops-demo
git add . && git commit -m "Commit Code" && git push
```

- Create your Cloud SQL MySQL Instance if you wanna test the app with a DB 
```
gcloud sql instances create petclinic-mysql-instance
gcloud sql databases create petclinic-db --instance petclinic-mysql-instance 
gcloud sql instances describe petclinic-mysql-instance | grep connectionName 
```

- Create Secrets using Google Secret Manager & update your application.propreties
```
spring.cloud.gcp.sql.instance-connection-name=${sm://petclinic-db-instance-connection-name}
spring.cloud.gcp.sql.database-name=petclinic-db
spring.datasource.username=${sm://petclinic-db-username}
spring.datasource.password=${sm://petclinic-db-password}
```

### Step 2-a : Build locally and Test

- Option A: Build and Test you app locally using Gradle
```
cd ~/projects/gcp-devops-demo
./gradlew build -x test  
./gradlew bootRun
curl localhost:8080
```

- Option B: Build and Test you app locally using Cloud Native Buildpacks and Docker!
```
./gradlew bootBuildImage --imageName=petclinic
docker images 
docker run -p 8080:8080 -t petclinic 
docker ps 
```

- Optional: Verifiy Data persistance 
```
gcloud sql connect petclinic-mysql-instance -u admin
mysql> use petclinic-db;
mysql> select * from owners;
```

### Step 2-b: Build your App using Cloud Build & Store it in Artifact Registry

- Creating the Artifact Registry Repository! 
```
gcloud artifacts repositories create petclinic-artifact-repo --repository-format=docker --location=us-central1 --description="Docker repo for petclinic" 
gcloud artifacts repositories list
```
- Building the App Using Cloud Build and Pushing it to Artifact Registry! 

```
nano cloudbuild.yaml // insert this code for the build and the push part 
```

```
steps:

# Build your app using Cloud Build and Buildpacks.io 
- name: 'gcr.io/k8s-skaffold/pack'
  entrypoint: 'pack'
  args: ['build', '--builder=gcr.io/buildpacks/builder', 
  '--publish', 'us-central1-docker.pkg.dev/YOUR_PROJECT_ID/petclinic-artifact-repo/petclinic:latest']
```

```
gcloud builds submit --config=cloudbuild.yaml
```
### Step 4: Deploy our App to Cloud Run using CloudBuild 
- Create a Service Account petclinic with Cloud SQL Client Permission! 
```
gcloud iam service-accounts create SERVICE_ACCOUNT_ID \
    --description="DESCRIPTION" \
    --display-name="DISPLAY_NAME" 

gcloud projects add-iam-policy-binding PROJECT_ID \
    --member="serviceAccount:SERVICE_ACCOUNT_ID@PROJECT_ID.iam.gserviceaccount.com" \
    --role="cloudsql.instances.connect"
```

- Edit your Cloudbuild.yaml to add the Deploy step! 
```
nano cloudbuild.yaml // insert this code to deploy the app using cloud build to cloud Run
```

```
# Deploy your app using Cloud Build to Google Cloud Run
- name: 'gcr.io/cloud-builders/gcloud'
  args: ['run', 'deploy', 'petclinic', 
  '--image', 'us-central1-docker.pkg.dev/YOUR_PROJECT_ID/petclinic-artifact-repo/petclinic:latest',
  '--region', 'us-central1', '--allow-unauthenticated', '--cpu=2', '--memory=512M',
  '--service-account', 'petclinic@YOUR_PROJECT_ID.iam.gserviceaccount.com', 
  '--update-secrets= petclinic-db-instance-connection-name=petclinic-db-instance-connection-name:latest',
  '--update-secrets= petclinic-db-name=petclinic-db-name:latest',
  '--update-secrets= petclinic-db-username=petclinic-db-username:latest',
  '--update-secrets= petclinic-db-password=petclinic-db-password:latest']
```

```
gcloud builds submit --config=cloudbuild.yaml
```

### Step 4: Automate your CI/CD with a Cloud Build Trigger!
```
gcloud beta builds triggers create cloud-source-repositories --repo=gcp-devops-demo --branch-pattern=master  --build-config=cloudbuild.yaml
git add . && git commit -m "Commit Code" && git push google master
```

### Step 5: Log, Monitor and Trace your Services
- To explore metrics, Logs and Error reporting! 
```
- https://console.cloud.google.com/run/detail/us-central1/petclinic/logs
- https://console.cloud.google.com/run/detail/us-central1/petclinic/metrics
- https://console.cloud.google.com/errors 
```

### Step 6 (Bonus): DevSecOps! Blocking docker images deployment outside Cloud Build 

- Create a project level policy to require Binary Authorization enforcement on images deployed to Cloud Run
```
gcloud resource-manager org-policies allow run.allowedBinaryAuthorizationPolicies \
  default \
  --project=YOUR_PROJECT_ID
```

- Second, use the built-by-cloud-build attestor to only deploy the images built by Cloud Build.
- Edit the Policy
- Add Attestors and require the built-by-cloud-build

- Run this to be able to pull or push inages from Artifact Registry
```
gcloud auth configure-docker us-central1-docker.pkg.dev
```

- Let's Build and tag the Petclinic image locally using Docker
```
./gradlew bootBuildImage --imageName=petclinic
docker tag petclinic us-central1-docker.pkg.dev/hamza-labs-1/petclinic/petclinic:bin-auth
```
- Push the bin-auth image to Artifact Registry
```
docker push us-central1-docker.pkg.dev/hamza-labs-1/petclinic/petclinic:bin-auth
```

- Deploy the new image to Cloud Run
```
gcloud run deploy petclinic --image us-central1-docker.pkg.dev/YOUR_PROJECT_ID/petclinic-artifact-repo/petclinic:bin-auth --region us-central1 --allow-unauthenticated --cpu=2 --memory=512M
  --service-account petclinic@YOUR_PROJECT_ID.iam.gserviceaccount.com --update-secrets= petclinic-db-instance-connection-name=petclinic-db-instance-connection-name:latest --update-secrets= petclinic-db-name=petclinic-db-name:latest --update-secrets= petclinic-db-username=petclinic-db-username:latest --update-secrets= petclinic-db-password=petclinic-db-password:latest
```

- For more about how to use Binary Authorization and Container registry scanning
```
- Go To https://cloud.google.com/binary-authorization
- Go To https://cloud.google.com/container-registry/docs/container-analysis
```

### Useful command line for cleaning 
```
docker rm -f $(docker ps -aq)
docker rmi $(docker images -a -q)
sudo usermod -a -G docker ${USER}
```