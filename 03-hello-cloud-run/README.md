````markdown
# Hello Cloud Run

## Overview

This lab demonstrates how to build and deploy a containerized Node.js application to **Cloud Run** using **Artifact Registry** and **Cloud Build**.

The implementation uses:

- **Cloud Shell** to develop and manage the application.
- **Node.js** and **Express** to build the web application.
- **Docker** and a **Dockerfile** to containerize the application.
- **Artifact Registry** to store the container image.
- **Cloud Build** to build the container image in the cloud.
- **Cloud Run** to deploy and serve the containerized application.

## Objectives

- Enable the required Google Cloud APIs.
- Create a Node.js application with Express.
- Create an Artifact Registry repository for Docker images.
- Write a Dockerfile to containerize the application.
- Build the container image using Cloud Build.
- Test the container image locally.
- Deploy the container to Cloud Run.
- Verify the running application through the Cloud Run service URL.

---

## Architecture

```text
Developer
    |
    | Application code
    v
Cloud Shell
    |
    | gcloud builds submit
    v
Cloud Build
    |
    | Container image
    v
Artifact Registry
    |
    | gcloud run deploy
    v
Cloud Run
    |
    | HTTPS
    v
User
````

---

# 1. Enable APIs and Configure Environment

The required Cloud Run and Artifact Registry APIs were enabled from Cloud Shell:

```bash
gcloud services enable run.googleapis.com artifactregistry.googleapis.com
```

The compute region was configured:

```bash
gcloud config set compute/region europe-west1
```

An environment variable was also created to store the selected region:

```bash
LOCATION="europe-west1"
```

The `gcloud` CLI was used to manage the Google Cloud resources from Cloud Shell.

![APIs Enabled and Environment Configured](screenshots/01-apis-enabled.png)

---

# 2. Create the Node.js Application

A directory for the application was created:

```bash
mkdir helloworld && cd helloworld
```

The application uses **Node.js** with the **Express** web framework.

The `package.json` file defines the application metadata, startup command, and Express dependency:

```json
{
  "name": "helloworld",
  "description": "Simple hello world sample in Node",
  "version": "1.0.0",
  "main": "index.js",
  "scripts": {
    "start": "node index.js"
  },
  "author": "Google LLC",
  "license": "Apache-2.0",
  "dependencies": {
    "express": "^4.17.1"
  }
}
```

The `index.js` file contains a simple Express web server:

```javascript
const express = require('express');
const app = express();
const port = process.env.PORT || 8080;

app.get('/', (req, res) => {
  const name = process.env.NAME || 'World';
  res.send(`Hello ${name}!`);
});

app.listen(port, () => {
  console.log(`helloworld: listening on port ${port}`);
});
```

### Application behavior

* Express creates the web server.
* The application listens on the port defined by the `PORT` environment variable.
* Port `8080` is used as the default.
* The `/` route returns `Hello World!`.
* `app.listen()` starts the server.

![Node.js Application](screenshots/02-nodejs-application.png)

---

# 3. Create an Artifact Registry Repository

A Docker repository named `my-repository` was created in Artifact Registry:

```bash
gcloud artifacts repositories create my-repository \
    --repository-format=docker \
    --location=$LOCATION \
    --description="Docker repository"
```

### Command explanation

* `gcloud artifacts repositories create`: Creates an Artifact Registry repository.
* `my-repository`: Name of the repository.
* `--repository-format=docker`: Specifies that the repository stores Docker images.
* `--location=$LOCATION`: Specifies the repository region.
* `--description`: Adds a description to the repository.

Docker authentication was then configured for Artifact Registry:

```bash
gcloud auth configure-docker $LOCATION-docker.pkg.dev
```

This allows Docker to authenticate when pulling or pushing images to the Artifact Registry Docker repository.

### Repository configuration

```text
Name:     my-repository
Format:   Docker
Location: europe-west1
```

![Artifact Registry Repository Created](screenshots/03-artifact-registry-repository.png)

---

# 4. Containerize the Application

A `Dockerfile` was created in the application directory. It defines the instructions required to build the container image:

```dockerfile
# Use the official lightweight Node.js image.
FROM node:20-slim

# Create and change to the app directory.
WORKDIR /usr/src/app

# Copy application dependency manifests to the container image.
COPY package*.json ./

# Install production dependencies.
RUN npm install --only=production

# Copy local code to the container image.
COPY . ./

# Run the web service on container startup.
CMD [ "npm", "start" ]
```

### Dockerfile instructions

| Instruction | Purpose                                                |
| ----------- | ------------------------------------------------------ |
| `FROM`      | Defines the base Node.js image                         |
| `WORKDIR`   | Sets the working directory inside the container        |
| `COPY`      | Copies application files into the image                |
| `RUN`       | Installs the production dependencies                   |
| `CMD`       | Defines the command executed when the container starts |

The Dockerfile acts as a recipe for creating the application container image.

![Dockerfile](screenshots/04-dockerfile.png)

---

# 5. Build the Container Image

The container image was built using **Cloud Build** and pushed directly to Artifact Registry:

```bash
gcloud builds submit \
    --tag $LOCATION-docker.pkg.dev/$GOOGLE_CLOUD_PROJECT/my-repository/helloworld
```

### Command explanation

* `gcloud builds submit`: Submits the application source to Cloud Build.
* `--tag`: Specifies the name and destination of the container image.
* `$LOCATION-docker.pkg.dev`: Artifact Registry Docker endpoint.
* `$GOOGLE_CLOUD_PROJECT`: Current Google Cloud project.
* `my-repository`: Artifact Registry repository.
* `helloworld`: Container image name.

Cloud Build builds the container image according to the Dockerfile and pushes the resulting image to Artifact Registry.

The build completed successfully:

```text
STATUS: SUCCESS
```

![Container Image Build](screenshots/05-container-image-build.png)

---

# 6. Verify the Container Image in Artifact Registry

After the successful build, the `helloworld` container image was available in the `my-repository` Docker repository.

The image was verified through:

**Artifact Registry → Repositories → my-repository → helloworld**

The image follows this structure:

```text
REGION-docker.pkg.dev/PROJECT_ID/REPOSITORY/IMAGE
```

For this lab:

```text
europe-west1-docker.pkg.dev/PROJECT_ID/my-repository/helloworld
```

![Container Image in Artifact Registry](screenshots/06-artifact-registry-image.png)

---

# 7. Test the Container Locally

Before deploying the application to Cloud Run, the container image was tested locally in Cloud Shell:

```bash
docker run -d -p 8080:8080 \
    $LOCATION-docker.pkg.dev/$GOOGLE_CLOUD_PROJECT/my-repository/helloworld
```

### Command explanation

* `docker run`: Creates and starts a container from the specified image.
* `-d`: Runs the container in detached mode.
* `-p 8080:8080`: Maps port `8080` on the host to port `8080` inside the container.
* The final argument specifies the container image.

The application was accessed using Cloud Shell **Web Preview** on port `8080`.

The expected response was:

```text
Hello World!
```

![Local Container Test](screenshots/07-local-container-test.png)

This confirms that the container starts correctly and that the Node.js application can handle HTTP requests.

---

# 8. Deploy the Application to Cloud Run

The container image was deployed to Cloud Run using:

```bash
gcloud run deploy helloworld \
    --image $LOCATION-docker.pkg.dev/$GOOGLE_CLOUD_PROJECT/my-repository/helloworld \
    --allow-unauthenticated \
    --region=$LOCATION
```

### Command explanation

* `gcloud run deploy`: Deploys an application to Cloud Run.
* `helloworld`: Name of the Cloud Run service.
* `--image`: Specifies the container image to deploy.
* `--allow-unauthenticated`: Allows public access to the service.
* `--region=$LOCATION`: Specifies the Cloud Run region.

Cloud Run created the service, created the first revision, and routed 100% of the traffic to the deployed revision.

### Deployment result

```text
Service:  helloworld
Revision: helloworld-00001-b2l
Region:   europe-west1
URL:      https://helloworld-904305763605.europe-west1.run.app
```

![Cloud Run Deployment](screenshots/08-cloud-run-deployment.png)

---

# 9. Verify the Application on Cloud Run

The Cloud Run service URL was opened in a browser:

```text
https://helloworld-904305763605.europe-west1.run.app
```

The deployed application successfully returned:

```text
Hello World!
```

![Application Running on Cloud Run](screenshots/09-cloud-run-application.png)

This confirms that the containerized application was successfully deployed and is accessible through a public HTTPS endpoint.

---

# 10. Verify the Cloud Run Service

The deployed service was also verified through:

**Google Cloud Console → Cloud Run → Services**

The service appears as:

```text
Service:          helloworld
Deployment type:  Container
Region:           europe-west1
```

![Cloud Run Service](screenshots/10-cloud-run-service.png)

---

# 11. Key Concepts

### Cloud Run

**Cloud Run** is a managed serverless platform for running stateless containers. It abstracts infrastructure management and automatically handles the deployment and scaling of container instances.

### Artifact Registry

**Artifact Registry** is a managed service for storing and managing container images and other software artifacts. In this lab, it stores the `helloworld` container image.

### Cloud Build

**Cloud Build** is a managed build service that builds the container image in Google Cloud using the Dockerfile and pushes the resulting image to Artifact Registry.

### Dockerfile

A **Dockerfile** contains the instructions used to build a container image. It defines the base image, working directory, application files, dependencies, and startup command.

### Container Image

A **container image** is a packaged version of an application and its required dependencies. The image created in this lab was stored in Artifact Registry and later deployed to Cloud Run.

### Serverless

Cloud Run is **serverless**, meaning Google Cloud manages the underlying infrastructure. The developer focuses on the application and container image instead of managing servers.

### Automatic Scaling

Cloud Run automatically adjusts the number of container instances according to incoming requests. This allows the application to scale based on demand.

### `--allow-unauthenticated`

This option makes the Cloud Run service publicly accessible without requiring authentication. It was used in this lab so the application could be accessed through its public HTTPS URL.

### `PORT` Environment Variable

The application uses the `PORT` environment variable:

```javascript
const port = process.env.PORT || 8080;
```

This allows the application to listen on the port provided by the Cloud Run environment.

---

# 12. Complete Commands Used

```bash
# Enable required APIs
gcloud services enable run.googleapis.com artifactregistry.googleapis.com

# Configure region
gcloud config set compute/region europe-west1

# Set location variable
LOCATION="europe-west1"

# Create application directory
mkdir helloworld && cd helloworld

# Create Artifact Registry repository
gcloud artifacts repositories create my-repository \
    --repository-format=docker \
    --location=$LOCATION \
    --description="Docker repository"

# Configure Docker authentication
gcloud auth configure-docker $LOCATION-docker.pkg.dev

# Build and push container image
gcloud builds submit \
    --tag $LOCATION-docker.pkg.dev/$GOOGLE_CLOUD_PROJECT/my-repository/helloworld

# Run the container locally
docker run -d -p 8080:8080 \
    $LOCATION-docker.pkg.dev/$GOOGLE_CLOUD_PROJECT/my-repository/helloworld

# Deploy to Cloud Run
gcloud run deploy helloworld \
    --image $LOCATION-docker.pkg.dev/$GOOGLE_CLOUD_PROJECT/my-repository/helloworld \
    --allow-unauthenticated \
    --region=$LOCATION
```

---

# 13. Cleanup

After completing the lab, the container image and Cloud Run service can be deleted to avoid keeping unnecessary resources.

### Delete the container image

```bash
gcloud artifacts docker images delete \
    $LOCATION-docker.pkg.dev/$GOOGLE_CLOUD_PROJECT/my-repository/helloworld
```

This removes the container image from Artifact Registry.

### Delete the Cloud Run service

```bash
gcloud run services delete helloworld \
    --region="REGION"
```

This removes the deployed Cloud Run service.

The cleanup step is important because storing container images in Artifact Registry can generate storage charges.

---

# 14. Technologies and Services

| Service / Technology  | Purpose                                          |
| --------------------- | ------------------------------------------------ |
| **Cloud Shell**       | Development and command-line environment         |
| **Node.js**           | JavaScript runtime for the application           |
| **Express**           | Web framework for Node.js                        |
| **Docker**            | Containerization platform                        |
| **Dockerfile**        | Instructions for building the container image    |
| **Cloud Build**       | Builds the container image in the cloud          |
| **Artifact Registry** | Stores the Docker container image                |
| **Cloud Run**         | Deploys and serves the containerized application |

---

# Conclusion

This lab provided practical experience with the complete workflow for deploying a containerized web application on Google Cloud.

The application was created with **Node.js and Express**, containerized using **Docker**, built using **Cloud Build**, stored in **Artifact Registry**, tested locally, and finally deployed to **Cloud Run**.

The final result was a containerized web application accessible through a public HTTPS endpoint managed by Cloud Run.

```
```
