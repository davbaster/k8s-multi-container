## Highlights of the Workflow:

1. Trigger on PR merge: Runs on push to google-cloud branch (which includes merged Pull Requests).

2. Keyless Authentication (Workload Identity Federation):

* Eliminates the security risk of generating and downloading a permanent JSON service account key.
* Uses google-github-actions/auth@v2 with permissions: id-token: write.

3. Automated Testing: Builds client/Dockerfile.dev and runs tests in CI mode.

4. Build & Versioning: Tags Docker images with both latest and ${{ github.sha }} for multi-client, multi-server, and multi-worker.

5. GKE Deployment:
* Acquires cluster credentials using google-github-actions/get-gke-credentials@v2.
* Runs kubectl apply -f k8s to ensure manifests are in sync.
* Runs kubectl set image ... to roll out the new SHA-tagged images without caching issues.

## Step-by-Step Setup in Google Cloud (To avoid using a Key File)

# Set your project variables based on your screenshot
Create a service account with Kubernetes admin role


Run these commands in Google Cloud Shell (or your terminal where gcloud is logged in):

# Set your project variables based on your screenshot
export PROJECT_ID="google project ide here"
export SA_EMAIL="google service account email (generated when service account was created)"
export GITHUB_REPO="username/yourreponame"

# 1. Enable required APIs
'''
gcloud services enable \
  iamcredentials.googleapis.com \
  container.googleapis.com \
  --project="${PROJECT_ID}"
'''

# 2. Create the Workload Identity Pool
'''
gcloud iam workload-identity-pools create "github-pool" \
  --project="${PROJECT_ID}" \
  --location="global" \
  --display-name="GitHub Actions Pool"
'''

# 3. Create the Workload Identity Provider
'''
gcloud iam workload-identity-pools providers create-oidc "github-provider" \
  --project="${PROJECT_ID}" \
  --location="global" \
  --workload-identity-pool="github-pool" \
  --display-name="GitHub Actions Provider" \
  --attribute-mapping="google.subject=assertion.sub,attribute.actor=assertion.actor,attribute.repository=assertion.repository" \
  --attribute-condition="assertion.repository=='${GITHUB_REPO}'" \
  --issuer-uri="https://token.actions.githubusercontent.com"

'''

# 4. Bind your service account to GitHub Actions repository
'''
export WORKLOAD_IDENTITY_POOL_ID=$(gcloud iam workload-identity-pools describe "github-pool" \
  --project="${PROJECT_ID}" \
  --location="global" \
  --format="value(name)")

'''
'''
gcloud iam service-accounts add-iam-policy-binding "${SA_EMAIL}" \
  --project="${PROJECT_ID}" \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/${WORKLOAD_IDENTITY_POOL_ID}/attribute.repository/${GITHUB_REPO}"
'''

# 5. Output the Workload Identity Provider resource name
'''
gcloud iam workload-identity-pools providers describe "github-provider" \
  --project="${PROJECT_ID}" \
  --location="global" \
  --workload-identity-pool="github-pool" \
  --format="value(name)"
'''


## GitHub Repository Secrets Needed

Add the following under Settings > Secrets and variables > Actions:

| Secret Name | Value | 
| --- | --- | 
| GCP_WORKLOAD_IDENTITY_PROVIDER | Output of command #5 above (e.g., projects/<PROJECT_NUMBER>/locations/global/workloadIdentityPools/github-pool/providers/github-provider) | 
| GCP_SERVICE_ACCOUNT | google service account email  |
| GKE_CLUSTER | Name of your GKE Cluster | 
| GKE_REGION | Compute region of your cluster (e.g., us-central1-a or us-central1) |
| DOCKER_USERNAME | Your Docker Hub username | 
| DOCKER_PASSWORD | Your Docker Hub personal access token or password |   

### Create secrets in kubernetes

Preparing the secret container in the kubernetes cluster:

'''
gcloud config set project <projectid>
'''

'''
gcloud config set compute/region <(e.g., us-central1-a or us-central1)>
'''

'''
gcloud container clusters get-credentials {cluster-name}
'''

Command to create the secret: 'kubectl create secret generic pgpassword --from-literal PGPASSWORD=12345asdf'

## Installing Helm

If you have Helm, you can deploy the ingress controller with the following command (Idempotent):

'''
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
'''

Check that your Ingress Controller was deployed:

kubectl get service ingress-nginx-controller --namespace=ingress-nginx