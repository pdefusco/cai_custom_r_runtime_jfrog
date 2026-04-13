# CAI Custom R Runtime with JFrog

## Objective

This demo shows how to create a custom R runtime extending the Cloudera AI R 4.5 PBJ Runtime, push it to a private Jfrog repository, and then import it into the CAI Runtime Catalog.

## End to End Demo

### 1. Create Dockerfile

In your local environment, create a new Dockerfile (if you cloned this github, an example has already been provided). Familiarize yourself with the code.

```
# Base Cloudera AI Runtime
FROM docker.repository.cloudera.com/cloudera/cdsw/ml-runtime-pbj-workbench-r4.5-standard:2026.01.1-b6

# Switch to root to install R packages if needed
USER root

# Install sparklyr without pulling Spark dependencies
# (sparklyr itself does not install Spark unless explicitly requested via spark_install())
RUN R -e "install.packages('sparklyr', repos='https://cloud.r-project.org', dependencies=TRUE)"

# Override Runtime label and environment variables metadata
ENV ML_RUNTIME_EDITOR="Workbench" \
    ML_RUNTIME_EDITION="Community" \
    ML_RUNTIME_SHORT_VERSION="2026.03" \
    ML_RUNTIME_MAINTENANCE_VERSION="2" \
    ML_RUNTIME_FULL_VERSION="2026.03.2" \
    ML_RUNTIME_DESCRIPTION="Runtime for Nikhil"

LABEL com.cloudera.ml.runtime.editor=$ML_RUNTIME_EDITOR \
      com.cloudera.ml.runtime.edition=$ML_RUNTIME_EDITION \
      com.cloudera.ml.runtime.full-version=$ML_RUNTIME_FULL_VERSION \
      com.cloudera.ml.runtime.short-version=$ML_RUNTIME_SHORT_VERSION \
      com.cloudera.ml.runtime.maintenance-version=$ML_RUNTIME_MAINTENANCE_VERSION \
      com.cloudera.ml.runtime.description=$ML_RUNTIME_DESCRIPTION
```

Notice the base CAI runtime "ml-runtime-pbj-workbench-r4.5-standard:2026.01.1-b6" is extended.

The CAI engineering team periodically tests, maintains and publishes this and other runtimes in [this GitHub repository](https://github.com/cloudera/ml-runtimes).

In this example, the "sparklyr" package is installed in the runtime so users who launch sessions, jobs, and other workloads in CAI don't have to.

You can customize this Dockerfile to include more packages. And, as shown in "sparklyrtest.R", you can still install other packages on top of the ones provided in the runtime. When you do this, package dependencies are saved in the project files so there is no need to reinstall in the future.

Notice Runtime Metadata is mandatory and must uniquely identify each build. For a more detailed explanation of the metadata fields, please visit [this page](https://docs.cloudera.com/machine-learning/cloud/runtimes/topics/ml-metadata-for-custom-runtimes.html) in the documentation.

### 2. Build and Push Image to JFrog

```
docker build -t cai-sparklyr-jfrog:latest .
```

```
docker tag cai-sparklyr-jfrog:latest \
  trialq7b92r.jfrog.io/cldr-demo-docker-local/cai-sparklyr-jfrog:latest
```

```
docker push \
  trialq7b92r.jfrog.io/cldr-demo-docker-local/cai-sparklyr-jfrog:latest
```

Validate build by going to "Artifactory" -> "Artifacts" -> "<name-of-your-repository>"

![alt text](img/jfrog-build.png)

### 3. Create Docker Credentials in Cloudera AI Workbench

Generate a Token in jfrog.

Navigate to "Site Administration" -> "Runtimes" -> "Docker Credentials" and create a new credential and set the following fields:

```
Name: sparklyr-jfrog-credentials
Server: the image build e.g. "trialq7b92r.jfrog.io/cldr-demo-docker-local/cai-sparklyr-jfrog:latest"
Username: <your-jfrog-username>
Password: <your-jfrog-token>
```

Notice that when creating the Docker Credentials, the value used in the form for the "Server" is the same value that was used from the docker cli to build the image, in this case ```trialq7b92r.jfrog.io/cldr-demo-docker-local/cai-sparklyr-jfrog:latest```. This is the same value used in the "Server" field when creating the Docker Credentials.

### 4. Import Runtime in the Catalog and Run Test Session

In the Runtime Catalog, select the "Add Runtime" icon.

Apply the previously configured credentials and input the entire URI path in the "Registry of Docker Image to Upload" field.

The image will now become available in the Runtime Catalog UI.

![alt text](img/creds-0.png)

![alt text](img/creds-1.png)

![alt text](img/runtime-catalog-1.png)

![alt text](img/runtime-catalog-2.png)

Notice that when adding the runtime to the Runtime Catalog, the value used in the form for the "Registry of Docker Image to Upload" is the same value that was used from the docker cli to build the image, in this case ```trialq7b92r.jfrog.io/cldr-demo-docker-local/cai-sparklyr-jfrog:latest```. This is the same value used in the "Server" field when creating the Docker Credentials.

![alt text](img/runtime-catalog-2.png)

### 5. Run Test Session

Clone this GitHub repository as a new CAI project and add the new runtime.

![alt text](img/project-runtimes-1.png)

Create a test session and run the code. Notice the URI is now showing as the Runtime ID in the Session UI.

```
Name: Sparklyr Test Session
Editor: Workbench
Kernel: R 4.5
Edition: Community
Version: 2026.03
Enable Spark: version 3.5
Enable GPU: none
Resource Profile: 2 vCPU / 4 GiB Memory
```

![alt text](img/session-1.png)

Next, run "sparklyrtest.R" and validate output.

![alt text](img/session-2.png)

![alt text](img/session-3.png)

## Summary and Next Steps

**Cloudera AI Runtimes** provide a structured way to package and manage reproducible environments for data science and machine learning workloads, enabling teams to define lightweight, customizable containers with specific editors, languages, and dependency stacks.

Runtimes are registered and managed in the Runtime Catalog, which lists all available standard and custom environments for interactive sessions or production workloads, and can be imported from Jfrog and other image registries.

Administrators and data scientists can create and import new runtimes in the catalog by adding custom images and credentials, enabling secure access to registries like Jfrog and on‑prem registries, and ensuring that the right tools and packages are available where and when teams need them. ([Cloudera Documentation][1])

### Cloudera AI Runtime Documentation & Blogs

* **Runtime Catalog documentation** – Official guide to viewing and managing the Runtime Catalog in Cloudera AI. ([Cloudera Documentation][1])
  *[https://docs.cloudera.com/machine-learning/1.5.5/runtimes/topics/ml-using-runtime-catalog.html](https://docs.cloudera.com/machine-learning/1.5.5/runtimes/topics/ml-using-runtime-catalog.html)*

* **Adding new ML Runtimes** – How to register custom ML Runtimes and add Docker registry credentials. ([Cloudera Documentation][2])
  *[https://docs.cloudera.com/machine-learning/cloud/managing-runtimes/topics/ml-adding-new-ml-runtimes.html](https://docs.cloudera.com/machine-learning/cloud/managing-runtimes/topics/ml-adding-new-ml-runtimes.html)*

* **Adding Docker registry credentials in CAI** – Instructions for adding Docker registry credentials (e.g., ECR) for pulling images. ([Cloudera Documentation][3])
  *[https://docs.cloudera.com/machine-learning/1.5.5/managing-runtimes/topics/ml-add-docker-registry-credentials-runtimes.html](https://docs.cloudera.com/machine-learning/1.5.5/managing-runtimes/topics/ml-add-docker-registry-credentials-runtimes.html)*

* **ML Runtimes overview and customization** – Full docs on ML Runtimes, editors, kernels, add‑ons, and custom images. ([Cloudera Documentation][4])
  *[https://docs.cloudera.com/machine-learning/cloud/runtimes/index.html](https://docs.cloudera.com/machine-learning/cloud/runtimes/index.html)*

* **Cloudera Blog: Building Custom Runtimes with Editors** – Example of customizing and using different editors in ML Runtimes. ([blog.cloudera.com][5])
  *[https://blog.cloudera.com/building-custom-runtimes-with-editors-in-cloudera-machine-learning/](https://blog.cloudera.com/building-custom-runtimes-with-editors-in-cloudera-machine-learning/)*
