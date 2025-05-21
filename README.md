# bdf-workbenches

## Project Goal

This repository provides everything needed to build a **custom RStudio workbench image for Red Hat OpenShift AI (RHOAI)**.  
The main goal is to deliver a ready-to-use RStudio environment with the [terra](https://cran.r-project.org/package=terra) and [sf](https://cran.r-project.org/package=sf) R libraries pre-installed, along with all required system dependencies.  
This image is designed for geospatial and spatial data science workloads on OpenShift AI, and can be extended with additional R or system packages as needed.

**Key features:**
- RStudio Server pre-installed and configured
- R `terra` and `sf` libraries (with all system dependencies)
- Compatible with OpenShift and Quay.io workflows
- Easily customizable for your own requirements

---

## Building and Deploying on OpenShift

### 0. Connect to Your OpenShift Cluster

Before building or deploying, log in to your OpenShift cluster using the `oc` CLI:

```sh
oc login <OPENSHIFT_API_URL> --token=<YOUR_TOKEN>
oc project <your-namespace>
```

Replace `<OPENSHIFT_API_URL>` and `<YOUR_TOKEN>` with your cluster's API endpoint and your access token.  
Set `<your-namespace>` to the project/namespace where you want to build and deploy.

---

### 1. Local Build and Push to Quay.io

You can build the image locally and push it to your Quay.io repository:

```sh
# Build the image locally
podman build -f c9s-python-3.11/Dockerfile.cpu -t quay.io/<username>/workbenches:rstudio-terra-fs-v2 .

# Log in to Quay.io (if not already)
podman login quay.io

# Push the image
podman push quay.io/<username>/workbenches:rstudio-terra-fs-v2
```

You can then deploy this image on OpenShift using a Deployment or DeploymentConfig that references the Quay.io image.

---

### 2. Remote Build on OpenShift (BuildConfig)

You can let OpenShift build the image from your GitHub repository and push it directly to Quay.io.

#### a. Create the BuildConfig

Apply the provided YAML (`manifest/build-rstudio.yaml`):

```sh
oc apply -f manifest/build-rstudio.yaml
```

#### b. Set Up Quay.io Push Credentials

If your Quay.io repository is private, create and link a secret:

```sh
oc create secret docker-registry quayio-secret \
  --docker-server=quay.io \
  --docker-username=<your-quay-username-or-robot> \
  --docker-password=<your-quay-password-or-token> \
  --docker-email=<your-email>
oc secrets link builder quayio-secret
```

#### c. Start the Build

```sh
oc start-build workbenches-rstudio-terra-fs-v2
```

#### d. Monitor the Build

```sh
oc logs -f bc/workbenches-rstudio-terra-fs-v2
```

#### e. Deploy the Image

Once the image is available in Quay.io, create a Deployment or DeploymentConfig in OpenShift that uses:

```yaml
image: quay.io/<username>/workbenches:rstudio-terra-fs-v2
```

---

### 3. Building and Pushing with Tekton Pipeline

You can automate the build and push process using a Tekton pipeline. This approach is recommended for CI/CD and reproducibility.

#### a. Prerequisites

- The OpenShift Pipelines (Tekton) Operator must be installed on your cluster.
- You have created the Tekton `Task`, `Pipeline`, and (optionally) `PipelineRun` YAML files (see `manifest/tekton-pipeline/`).

#### b. Apply the Tekton Resources

Apply the tasks and pipeline (do **not** apply the PipelineRun yet if you don't want to run immediately):

```sh
oc apply -f manifest/tekton-pipeline/tasks/git-clone.yaml
oc apply -f manifest/tekton-pipeline/tasks/buildah-build.yaml
oc apply -f manifest/tekton-pipeline/tasks/buildah-push.yaml
oc apply -f manifest/tekton-pipeline/pipeline/build-and-push-rstudio-pipeline.yaml
```

#### c. Run the Pipeline

To start the pipeline, apply the PipelineRun YAML:

```sh
oc apply -f manifest/tekton-pipeline/pipeline-run/pipeline-run.yaml
```

Or create a PipelineRun from the web console or CLI.

#### d. Monitor the Pipeline

```sh
tkn pipelinerun logs -f <pipeline-run-name>
```
or use the OpenShift web console.

#### e. Notes

- The pipeline will clone your repo, build the image using Buildah, and push it to Quay.io.
- Make sure your service account has the necessary permissions and secrets to push to Quay.io.
- You can customize the pipeline parameters (git repo, context dir, image name, etc.) in the PipelineRun YAML.

---

### Notes

- The build context is set to `c9s-python-3.11` and uses `Dockerfile.cpu`.
- If the build fails to push to Quay.io, ensure your credentials are correct and linked as a secret.
- For troubleshooting, check build logs with `oc logs -f bc/<buildconfig-name>` or `tkn pipelinerun logs -f <pipeline-run-name>`.