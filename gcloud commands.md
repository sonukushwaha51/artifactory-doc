# Gcloud command for creating artifact registry
`gcloud artifacts repositories create my-maven-repo \
  --repository-format=maven \
  --location=us-central1 \
  --description="Maven repo for Java artifacts"`

# Grant service account access to read/write to artifact registry
`gcloud artifacts repositories add-iam-policy-binding my-maven-repo \
  --location=us-central1 \
  --member="serviceAccount:YOUR_SA@gcp-project.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.writer"`

# 
