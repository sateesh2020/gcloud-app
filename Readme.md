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

