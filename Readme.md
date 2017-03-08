## Installation and Setup

### Install Docker

### Install Google Cloud 

gcloud is a command-line tool for managing resources on Google Cloud Platform and is provided as part of Google Cloud SDK. To install, download the latest version [here](https://cloud.google.com/sdk/docs/#windows) and extract it to your local system. Ensure that you have Python 2.7 installed.

Once downloaded, extract the folder and run the following command to install gcloud and add it to your path.

#### Check GCLOUD Configuratin

```sh

gcloud config list

```

#### KUBECTL

Kubernetes provides a number of ways to install the kubectl command used to interact with it on your local system. The easiest approach to setup is installing it as part of Google Cloud SDK.

```sh

gcloud components install kubectl

```

### Creating GCP Project

Every Google project begins with creating a project on it's dashboard. Once you have your subscription or trial account ready, head over to https://console.cloud.google.com to create your first project.

Click on create new project and give it a memorable name. Take note of the project ID as we will be using it across the entire process.

This may take a while to finish. Once it's done, click on the Products & Services icon and select Container Engine which is where our Docker images will live.

The Container clusters manages our Google Compute Engine instances also reffered to as nodes.

The Container Registry is a repository for all our Docker images from which containers will be created.

### Preparing our Deployment Environment

#### SET DEFAULT PROJECT

```sh

gcloud config set project project-id

```

#### CLUSTERS

A cluster is a group of Google Compute Engine instances running Kubernetes. Pods and Services which we will be looking at later on, all run on top of a cluster.

##### Creating Cluster

You can create a cluster on the Container Clusters section on the project dashboard by going to Container Engine > Container Clusters > New container cluster or via the command-line using the kubectl command. Simply run

```sh
gcloud container clusters create cluster-id
```

The cluster may take some time to complete. Once it's done, again, set it as the default cluster.

```sh
gcloud config set container/cluster cluster-id
```

#### Getting Cluster Credentials

We now have a Kubernetes cluster on Our GCP project that will contain our containerized application. Let's notify our local setup that we are authorized to use the cluster.

On clicking the connect button shown in the cluster above, you will be presented with the command to configure Kubernetes with the newly created scotch-cluster.

The commnds will look something like this

```sh
gcloud container clusters get-credentials cluster-id --zone us-central1-a --project project-id

kubectl proxy

```

### A Look At Our NodeJS App.

Our app is going to be very simple and straight forward. We are going to have a single index.js file that serves 3 routes using express.

```sh
├── .gitignore
├── package.json
├── index.js
├── Dockerfile
├── deployment.yml
├── service.yml
```

**server.js**

```js
const app = require('express')();

const port = process.env.PORT || 8000;

app.get('/', (req, res) => {
  res.status(200).json('welcome to root');
});

app.get('/foo', (req, res) => {
  res.status(200).json('welcome to foo');
});

app.get('/bar', (req, res) => {
  res.status(200).json('welcome to bar');
});

app.listen(port, () => console.log('Magic happens on port', port));

module.exports.app = app;
```

### Dockerizing Our Node App
Make sure you have Docker running on your local setup before proceeding. While we will not go to the details of how Docker works, take a look at this article on [getting started with Docker](https://scotch.io/tutorials/getting-started-with-docker).

**Dockerfile**

```yaml
#Create our image from Node 6.9-alpine
FROM node:6.9-alpine

MAINTAINER John Kariuki <johnkariukin@gmail.com>

#Create a new directory to run our app.
RUN mkdir -p /usr/src/app

#Set the new directory as our working directory
WORKDIR /usr/src/app

#Copy all the content to the working directory
COPY . /usr/src/app

#install node packages to node_modules
RUN npm install

#Our app runs on port 8000. Expose it!
EXPOSE 8000

#Run the application.
CMD npm start

```

#### CREATING A DOCKER IMAGE

To create our Google Cloud platform-ready Docker image, we have to abide by some rules before it is pushed to the Google Container Registry. The image has to be created in the following format. gcr.io/{$project_id}/{image}:{tag}
You can also use the image from your personal docker registry.

```sh

$ docker build -t gcr.io/app-name-155622/node-app:0.0.1 .

```

#### PUSHING THE DOCKER IMAGE TO CONTAINER REGISTRY

With our Docker image ready, pushing it to the Container Registry is simply done with the gcloud command.

```sh
gcloud docker -- push gcr.io/app-name-155622/node-app:0.0.1
```

This may take a while but once it's done, we should have the image listed on Compute > Container Engine > Container Registry. which can be managed/orchestrated by Kubernetes!

### Deployments: Instantiating A Container From the Docker Image

All we have now is a Docker image on the Container Registry. To actually run the application, we need to create an instance from the image - A Docker container.

To achieve this, we will create a Kubernetes Deployment which creates a pod for our container.

*A Kubernetes pod is a group of containers, tied together for the purposes of administration and networking that are always co-located and co-scheduled, and run in a shared context.*

Containers within a pod share an IP address and port space, and can find each other via localhost

There are two ways to create a deployment using the kubectl command. You can either specify the parameters in a yml file or the command-line.

**Command-line approach**

The deployment name is a unique identifier for your deployment that will be used to reference it later on. Specify the image name to create the deployment from and finally a port which will be mapped to our applications port that is exposed (see Dockerfile).

```sh

kubectl run {deployment_name} --image=gcr.io/$PROJECT_ID/{name}:{tag} --port={port}

```

**file approach**

I personally prefer this approach as it is easier for me to play around with the configurations of the deployment. Here is a look at our Deployment file, subtly named *deployment.yml*

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: scotch-dep
  labels:
    #Project ID
    app: scotch-155622
spec:
  #Run two instances of our application
  replicas: 2
  template:
    metadata:
      labels:
        app: scotch-155622
    spec:
      #Container details
      containers:
        - name: node-app
          image: gcr.io/scotch-155622/node-app:0.0.1
          imagePullPolicy: Always
          #Ports to expose
          ports:
          - containerPort: 8000
```
One of the interesting aspects is that deployments allows you to set the number of instances for your application. This scaling Kubernetes provision allows you to add or remove instances/replicas/pods depending on the traffic needs of your app.

To create the deployment, run:

```sh

kubectl create -f deployment.yml

```

You can view the deployment and the created pods by running *kubectl get deployments* and *kubectl get pods* respectively. Adding a *-w* enables us to see how the pods are created.

### Services: Expose application to External traffic

Up until this point, there has been no way for us to actually run the application externally. Currently, the deployment is only accessible within the Kubernetes cluster (say, between two services in the same cluster).

Pods, like containers, can also be created and destroyed anytime, say, when scaling up or down or when rolling to an updated image. This means we cannot rely on the internal IPs since we cannot keep track of them.

*A Kubernetes Service is an abstraction which defines a logical set of Pods and a policy by which to access them - sometimes called a micro-service. It enables external traffic exposure, load balancing and service discovery for those Pods.*

Just like a deployment, You can create a service from a them command-line or from a file.

**command-line approach**

```sh
kubectl expose deployment {service_name} --type="LoadBalancer"
```

**file approach**

Here is our Service file, again, subtly name *service.yml*.

```yaml
kind: Service
apiVersion: v1
metadata:
  #Service name
  name: node-app-svc
spec:
  selector:
    app: scotch-155622
  ports:
    - protocol: TCP
      port: 8000
      targetPort: 8000
  type: LoadBalancer
```

The Kubernetes master creates the load balancer and related Compute Engine forwarding rules, target pools, and firewall rules to make the service fully accessible from outside of Google Cloud Platform.

To create the service, run:

```sh
kubectl create -f service.yml
```

Once the service is created, we can get the externally accessible IP address by listing all the services (*kubectl get services*). The external IP may take a few seconds to be visible.

### Scaling Up/Down your Application

Like we mentioned, one of the best provisions of Kubernetes is that you can scale your application up and down depending on your traffic requirements. This can be done by running the following command:

```sh
kubectl scale deployment {deployment_name} --replicas=n
```

In our case, the deployment name is *app-name* and n can be 3 replicas to scale up from 2 instances or 1 to scale down. You can actually see the pods come alive or get terminated by running the *kubectl get pods -w* command.

**AUTOSCALING**

You can also enable autoscaling in the cluster and set the minimum and maximum number of Pods based on the CPU utilization from the existing pods

```sh
kubectl autoscale deployment nginx-deployment --min=5 --max=10 --cpu-percent=75
```

### Kubernetes Web UI

Kubernetes has a very neat web User Interface for every cluster that makes it easy to interact with the Kubernetes functionality - From deploying to autoscaling to managing and monitoring clusters.

To access the UI locally, run the following command.

```sh
$ kubectl proxy
```

Go to [http://localhost:8001/ui](http://localhost:8001/ui), you should see a dashboard. This particular service section displays all the pods running in the cluster.


