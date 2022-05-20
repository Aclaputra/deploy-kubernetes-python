## Prepare the Quiz Application
### Clone source code in Cloud Shell
Click Open Terminal and clone the repository for the lab.
```bash
git clone https://github.com/GoogleCloudPlatform/training-data-analyst
```
Create a soft link as a shortcut to the working directory.
```bash
ln -s ~/training-data-analyst/courses/developingapps/v1.2/python/kubernetesengine ~/kubernetesengine
```
### Configure the Quiz application
Change the directory that contains the sample files for this lab.
```bash
cd ~/kubernetesengine/start
```
Configure the Quiz application.
```bash
. prepare_environment.sh
```
This script file:
- Creates a Google App Engine application.
- Exports environment variables GCLOUD_PROJECT and GCLOUD_BUCKET.
- Updates pip then runs pip install -r requirements.txt.
- Creates entities in Google Cloud Datastore.
- Creates a Google Cloud Pub/Sub topic.
- Creates a Cloud Spanner Instance, Database, and Table.
- Prints out the Project ID.

In Cloud Shell, make sure you are in the start folder:
```bash
Creating Cloud Pub/Sub topic
Created topic [projects/qwiklabs-gcp-92b7e5716e0cbf7e/topics/feedback].
Created subscription [projects/qwiklabs-gcp-92b7e5716e0cbf7e/subscriptions/worker-subscription].
Creating Cloud Spanner Instance, Database, and Table
Creating instance...done.
Creating database...done.
Project ID: qwiklabs-gcp-92b7e5716e0cbf7e
```
## Review the code
### Examine the code
Navigate to training-data-analyst/courses/developingapps/v1.2/python/kubernetesengine/start.

The folder structure for the Quiz application reflects how it will be deployed in Kubernetes Engine.

The web application is in a folder called frontend.

The worker application code that subscribes to Cloud Pub/Sub and processes messages is in a folder called backend.

There are configuration files for Docker (a Dockerfile in the frontend and backend folder) and backend-deployment and frontend-deployment Kubernetes Engine .yaml files.

## Create and connect to a Kubernetes Engine Cluster
### Create a Kubernetes Engine Cluster
- In the Cloud Platform Console, click Navigation menu > Kubernetes Engine > Clusters.
- Click CREATE.
- CONFIGURE the Standard cluster. Set the following fields to the provided values, leave all others at the default value:
Name : quiz-cluster, Zone: us-central1-b, default Pool > Security > Access scopes: Select Allow full access to all Cloud APIs
- Click CREATE. The cluster takes a few minutes to provision.

### Connect to the cluster
In this section you connect the Quiz application to the kubernetes cluster.
- When the cluster is ready, click on the Actions icon and select Connect.
- In Connect to the cluster, click Run in Cloud Shell to populated Cloud shell with the command that resembles gcloud container clusters get-credentials quiz-cluster --zone us-central1-b --project [Project-ID]. Press Enter to run the commnand in Cloud Shell.
- Run the following command to list the pods in the cluster:
```bash
kubectl get pods
```
## Build Docker Images using Container Builder
In this section, you create a Dockerfile for the application frontend and backend, and then employ Container Builder to build images and store them in the Container Registry.
### Create the Dockerfile for the frontend and backend
In the Cloud Shell code editor, open frontend/Dockerfile. You will now add a block of code that does the following:
- Enters the Dockerfile command to initialize the creation of a custom Docker image using Google's Python - App Engine image as the starting point.
- Writes the Dockerfile commands to activate a virtual environment.
- Writes the Dockerfile command to execute pip install as part of the build process.
- Writes the Dockerfile command to add the contents of the current folder to the /app path in the container.
- Completes the Dockerfile by entering the statement, gunicorn ..., that executes when the container runs. Gunicorn (Green Unicorn) is an HTTP server that supports the Python Web Server Gateway Interface (WSGI) -  specification.

Copy and paste the following to Dockerfile:
```bash
FROM gcr.io/google_appengine/python
RUN virtualenv -p python3.7 /env
ENV VIRTUAL_ENV /env
ENV PATH /env/bin:$PATH
ADD requirements.txt /app/requirements.txt
RUN pip install -r /app/requirements.txt
ADD . /app
CMD gunicorn -b 0.0.0.0:$PORT quiz:app
```
Open the backend/Dockerfile file and copy and paste the following code:
```bash
FROM gcr.io/google_appengine/python
RUN virtualenv -p python3.7 /env
ENV VIRTUAL_ENV /env
ENV PATH /env/bin:$PATH
ADD requirements.txt /app/requirements.txt
RUN pip install -r /app/requirements.txt
ADD . /app
CMD python -m quiz.console.worker
```
Build Docker images with Container Builder

In Cloud Shell, make sure you are in the start folder:
```bash
cd ~/kubernetesengine/start
```
Run the following command to build the frontend Docker image:
```bash
gcloud builds submit -t gcr.io/$DEVSHELL_PROJECT_ID/quiz-frontend ./frontend/
```
The files are staged into Cloud Storage, and a Docker image is built and stored in the Container Registry. It takes a few minutes.

Ignore any incompatibility messages you see in the output messages.\

Now run the following command to build the backend Docker image:
```bash
gcloud builds submit -t gcr.io/$DEVSHELL_PROJECT_ID/quiz-backend ./backend/
```
When the backend Docker image is ready you see these last messages:
```bash
DONE
-----------------------------------------------------------------------------------------------------------------------
ID                                    CREATE_TIME                DURATION  SOURCE                                      
                                                             IMAGES                                                    
   STATUS
be0326f4-3f6f-42d6-850f-547e260dd4d7  2018-06-13T22:20:16+00:00  50S       gs://qwiklabs-gcp-3f89d0745056ee31_cloudbuil
d/source/1528928414.79-4914d2a972f74e188f40ced135662b7d.tgz  gcr.io/qwiklabs-gcp-3f89d0745056ee31/quiz-backend (+1 more
)  SUCCESS
```
In the Cloud Platform Console, from the Navigation menu menu, click Container Registry. You should see two pods: quiz-frontend and quiz-backend.

Click quiz-frontend.

You should see the image Name and Tags (latest).

## Create Kubernetes Deployment and Service Resources
In this section, you will modify the template yaml files that contain the specification for Kubernetes Deployment and Service resources, and then create the resources in the Kubernetes Engine cluster.
### Create a Kubernetes Deployment file
In the Cloud Shell code editor, open the frontend-deployment.yaml file.

The file skeleton has been created for you. Your job is to replace placeholders with values specific to your project.

### Replace the placeholders in the frontend-deployment.yaml file using the following values:
- [GCLOUD_PROJECT] = Project ID(Display the Project ID by enteringecho $GCLOUD_PROJECT in Cloud Shell)
- [GCLOUD_BUCKET] = Cloud Storage bucket name for the media bucket in your project(Display the bucket name by enteringecho $GCLOUD_BUCKET in Cloud Shell)
- [FRONTEND_IMAGE_IDENTIFIER] = The frontend image identified in the form gcr.io/[Project_ID]/quiz-frontend

The quiz-frontend deployment provisions three replicas of the frontend Docker image in Kubernetes pods, distributed across the three nodes of the Kubernetes Engine cluster.

Save the file.

Replace the placeholders in the backend-deployment.yaml file using the following values:
- [GCLOUD_PROJECT] = Project ID
- [GCLOUD_BUCKET] = Cloud Storage bucket ID for the media bucket in your project
- [BACKEND_IMAGE_IDENTIFIER] = The backend image identified in the form gcr.io/[Project_ID]/quiz-backend

The quiz-backend deployment provisions two replicas of the backend Docker image in Kubernetes pods, distributed across two of the three nodes of the Kubernetes Engine cluster.

- Save the file.

- Review the contents of the frontend-service.yaml file.

The service exposes the frontend deployment using a load balancer. The load balancer sends requests from clients to all three replicas of the frontend pod.

### Execute the Deployment and Service Files
- In Cloud Shell, provision the quiz frontend Deployment.
```bash
kubectl create -f ./frontend-deployment.yaml
```
- Provision the quiz backend Deployment.
```bash
kubectl create -f ./backend-deployment.yaml
```
- Provision the quiz frontend Service.
```bash
kubectl create -f ./frontend-service.yaml
```

Each command provisions resources in Kubernetes Engine. It takes a few minutes to complete the process.

## Test the Quiz Application
In this section you review the deployed Pods and Service and navigate to the Quiz application.
### Review the deployed resources
- In the Cloud Console, select Navigation menu > Kubernetes Engine.
- Click Workloads in the left menu.
You should see two containers: quiz-frontend and quiz-backend. You may see that the status is OK or in the process of being created.

If the status of one or both containers is Does not have minimum availability, refresh the window.
- Click quiz-frontend. In the Managed pods section, there are three quiz-frontend pods.
- In the Exposing services section near the bottom, find the Endpoints section and copy the the IP address and paste it into the URL field of a new browser tab or window:
- This opens the Quiz application, which means you successfully deployed the application! You can end your lab here or use the remainder of the time to build some quizzes.

[>sumber belajar<](https://www.cloudskillsboost.google/catalog_lab/979)