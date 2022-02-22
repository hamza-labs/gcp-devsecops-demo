# Welcome to this quick GCP DevSecoPs Demo! 
<img src="https://github.com/hamza-labs/gcp-devops-demo/blob/main/src/main/resources/static/resources/images/devsecops.png?raw=true" width="650">

### Before you begin, you will need this! 

- To demonstrate how we can secure our software supply chain on GCP, we will rely on this repo (gcp-devops-demo)
- After deploying the demo app, we will use the same DevOps Toolchain.  

### Step 1: Explore how you can secure your app at the Build Stage

- https://cloud.google.com/artifact-registry/docs/analysis

### Step 2: Explore how you can secure your app at the Deploy Stage

- Run this to be able to pull or push inages from Artifact Registry
```
gcloud auth configure-docker us-central1-docker.pkg.dev
```

- Tag the Petclinic image, and Push to Artifact Registry 
```
docker tag petclinic us-central1-docker.pkg.dev/hamza-labs-1/petclinic/petclinic:bin-auth
docker push us-central1-docker.pkg.dev/hamza-labs-1/petclinic/petclinic:bin-auth

- Deploy the new image to Cloud Run
```
gcloud run deploy petclinic-bin-auth --image us-central1-docker.pkg.dev/hamza-labs-1/petclinic/petclinic:bin-auth \
--platform managed --region us-central1 --allow-unauthenticated \
--service-account petclinic1@hamza-labs-1.iam.gserviceaccount.com --add-cloudsql-instances mysql-instance  --cpu=2 --memory=512M
```

- For more about how to use Binary Authorization and Container registry scanning
```
- Go To https://cloud.google.com/binary-authorization
- Go To https://cloud.google.com/container-registry/docs/container-analysis
```
