# artifactory-doc

To push your Maven and Gradle project JARs to Google Cloud Artifact Registry (GAR) via GitHub Actions and use them as dependencies, you need to set up GCP resources, configure authentication, modify your project build files, and create a GitHub Actions workflow. 

# Phase 1: Google Cloud Setup
Enable APIs: In your GCP project, enable the Artifact Registry API and the IAM Service Account Credentials API.
Create a Repository: Create a Maven-format repository in Artifact Registry.

`gcloud artifacts repositories create my-java-repo --repository-format=maven --location=YOUR_REGION --description="Maven repository for Java packages"`


# Create a Service Account: This account will be used by GitHub Actions to authenticate with GCP.

`gcloud iam service-accounts create github-actions-sa --display-name="GitHub Actions SA"`


# Grant IAM Roles: The service account needs permissions to publish and download artifacts. The roles/artifactregistry.createOnPushWriter role (or Artifact Registry Writer) and roles/artifactregistry.reader role are appropriate.

`gcloud artifacts repositories add-iam-policy-binding my-java-repo --location=YOUR_REGION --member="serviceAccount:github-actions-sa@YOUR_PROJECT_ID.iam.gserviceaccount.com" --role="roles/artifactregistry.writer"`

# Set up Workload Identity Federation (Recommended): This is the more secure, keyless authentication method. It links your GitHub repository to the GCP service account.

Create a Workload Identity Pool and a GitHub Identity Provider.
Bind the service account to the identity provider, restricting access to your specific GitHub organization and repository. The GCP documentation provides specific gcloud commands for this process.
Once set up, you'll have a WORKLOAD_IDENTITY_PROVIDER ID for use in GitHub Actions. 

# Phase 2: GitHub Setup
Add GitHub Secrets: In your GitHub repository settings, add secrets:
`GCP_PROJECT_ID: Your GCP project ID.`

`GCP_ARTIFACT_REGISTRY_LOCATION: The region where your repository is located (e.g., us-central1).`

`GCP_ARTIFACT_REGISTRY_REPO: The name of your repository (e.g., my-java-repo).`

`WORKLOAD_IDENTITY_PROVIDER and GCP_SERVICE_ACCOUNT (if using Workload Identity Federation).`

# Phase 3: Project Configuration

You will need to configure your Maven and Gradle projects to know where to deploy the JARs and from where to download dependencies. The repository URL will look like: https://YOUR_REGION-maven.pkg.dev/YOUR_PROJECT_ID/my-java-repo. 
Maven Projects
pom.xml: Add the <distributionManagement> section to define where to deploy artifacts.

`<distributionManagement>
    <repository>
        <id>artifact-registry</id>
        <url>https://YOUR_REGION-maven.pkg.dev/YOUR_PROJECT_ID/my-java-repo</url>
    </repository>
    <snapshotRepository>
        <id>artifact-registry</id>
        <url>https://YOUR_REGION-maven.pkg.dev/YOUR_PROJECT_ID/my-java-repo</url>
    </snapshotRepository>
</distributionManagement>`


pom.xml (for consuming dependencies): To download dependencies, add the <repositories> section.

`<repositories>
    <repository>
        <id>artifact-registry</id>
        <url>https://YOUR_REGION-maven.pkg.dev/YOUR_PROJECT_ID/my-java-repo</url>
        <releases>
            <enabled>true</enabled>
        </releases>
        <snapshots>
            <enabled>true</enabled>
        </snapshots>
    </repository>
</repositories>`

 
Gradle Projects
build.gradle (or build.gradle.kts): In the publishing block, define the maven repository.
`gradle
publishing {
    repositories {
        maven {
            url "https://YOUR_REGION-maven.pkg.dev/YOUR_PROJECT_ID/my-java-repo"
        }
    }
}`

build.gradle (for consuming dependencies): In the repositories block, add the GAR URL.
`gradle
repositories {
    maven {
        url "https://YOUR_REGION-maven.pkg.dev/YOUR_PROJECT_ID/my-java-repo"
    }
}`

 
# Phase 4: GitHub Actions Workflow 
Create a workflow file (e.g., .github/workflows/publish.yaml) to automate the build and deployment. The key steps involve authenticating to GCP using the service account and running the build/publish commands. 

`name: Publish to GCP Artifact Registry

on:
  push:
    branches:
      - main

jobs:
  build-and-publish:
    runs-on: ubuntu-latest
    permissions:
      contents: 'read'
      id-token: 'write' # Required for Workload Identity Federation
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
      - name: Google Auth
        id: auth
        uses: google-github-actions/auth@v2
        with:
          project_id: ${{ secrets.GCP_PROJECT_ID }}
          workload_identity_provider: ${{ secrets.WORKLOAD_IDENTITY_PROVIDER }}
          service_account: ${{ secrets.GCP_SERVICE_ACCOUNT }}
      - name: Setup Java
        uses: actions/setup-java@v4
        with:
          java-version: '17' # Use your desired Java version
          distribution: 'temurin'
          # For Maven
          server-id: artifact-registry 
          # For Gradle (if needed)
          # username: _json_key
          # password: ${{ secrets.GCP_SERVICE_ACCOUNT_KEY }} 
     - name: Configure Maven for Artifact Registry
        run: |
          gcloud artifacts print-settings mvn --project=${{ secrets.GCP_PROJECT_ID }} --repository=${{ secrets.GCP_ARTIFACT_REGISTRY_REPO }} --location=${{ secrets.GCP_ARTIFACT_REGISTRY_LOCATION }} >> settings.xml
          # Append the content of settings.xml to the global Maven settings file
          cat settings.xml >> ~/.m2/settings.xml
     - name: Build and Publish Maven packages
        run: mvn clean deploy -s ~/.m2/settings.xml
`

The gcloud artifacts print-settings mvn command generates the necessary repository configuration with temporary credentials, which is then added to the Maven settings.xml file for the build to use. The actions/setup-java action helps configure the settings.xml with the server ID defined in your pom.xml. 
This approach enables you to automatically publish and manage your project artifacts in Artifact Registry. 

