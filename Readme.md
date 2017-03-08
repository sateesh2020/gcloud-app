# Google Cloud Platform II: Continuously Deploying A Docker Application On Google Container Engine

We previously looked at how to deploy a Docker application to Google Cloud Platform (GCP) with Google Container Engine(GKE) using Kubernetes. [TutorialOne](https://github.com/sateesh2020/gcloud-app/tree/tutorialone)

This gave us the opportunity to appreciate the role that Kubernetes plays in managing/orchestrating GKE clusters with ease.

One thing that we missed out on was how deploying Docker containers pans out when working in a team that employs Continuous Integration / Continuous Deployment (CI/CD) where frequent updates of an application are made by each member of the team and, well, continously deployed to staging or production.

As you would imagine, things can quickly get out of hand when each team member individually pushes their own version of the image to the Google Container Registry.

The image tags may conflict, be out of date or buggy images may be pushed to production and the world as we know it would come to a stop.

## Continuous Deployment with CircleCI

CircleCI[https://circleci.com/] is one of the most popular CI/CD tools in use today alongside TravisCI and Codeship. We wil be using CircleCI to continuously deploy our [application](https://github.com/sateesh2020/gcloud-app) just like we would in a distributed team.

To achieve this, we will do the following:

+ Write tests for our application.
+ Configure CircleCI to authenticate with our GKE cluster.
+ Add a *circle.yml* configuration file for the application deployment process.
+ If the tests pass on CircleCI, update the containerized app to GKE.

### Adding Tests

We'll use supertest to test the routes in index.js in the test and mocha test framework to run the test.

```sh
npm install --save-dev mocha supertest
```

Once installed, create *test/index_test.js* in the application route and add the test below.

```js
//Load supertest and the app instance.
const request = require('supertest');
const app = require('../index').app;

describe('Index Tests', () => {
    describe('GET /', () => {
        //Should return status code 200 with JSON message: welcome to root
        it('Should return root message', (done) => {
            request(app)
                .get('/')
                .expect(200)
                .expect(JSON.stringify('welcome to root'))
                .end(done);
        });
    });
});
```

The test makes a GET request to the root of our project and asserts that a JSON string is given back with a 200 status code. Add the test command in the scripts section of our package,json file.

```json
...
"scripts": {
    "test": "node_modules/.bin/mocha",
    "start": "node index.js"
  },
...

```

View the test result by running npm test. You should get the a pass result

### INTEGRATING CIRCLECI

With our tests passing, let's add a *circle.yml* file and instruct CircleCI how to run our tests using mocha. Luckily, every CircleCI build comes preinstalled with mocha so all we need to do is instruct it to use mocha.

Note that by default, CircleCI will try to run *npm test*.

```yaml
test:
    override:
    - mocha
```

### ADD PROJECT TO CIRCLECI

Once everything is set up, push the code to a Gihub repository, login to CircleCI with Github then go to CircleCI > Add Projects and build your repository. If everything is correctly set up, you should have a succesful build.

## Authenticating The GKE Cluster on CircleCI

The objective of this section is to simply let CircleCI know that we own the *cluster-1*. When creating the Deployments and Services locally, we followed the following protocol:

+ Login to Google - Using the *gcloud init* in the initial setup or *gcloud auth login* later on.
+ Set the default project - *gcloud config set project node-app-160805*
+ Set the default cluster - *gcloud config set container/cluster cluster-1*
+ Get the cluster credentials - *gcloud container clusters get-credentials cluster-1 --zone us-central1-a --project node-app-160805*
+ Gone ahead to be awesome.

But how do we authenticate CircleCI? We cannot exactly use *gcloud auth login* can we? That would mean allowing access to our gmail accounts. Sounds wrong, doesn't it?

And it is! Gladly, Google Cloud Platform provides [Service Accounts](https://cloud.google.com/compute/docs/access/service-accounts) which come in handy.

*A service account is a special account that can be used by services and applications running on your Google Compute Engine instance to interact with other Google Cloud Platform APIs. Applications can use service account credentials to authorize themselves to a set of APIs and perform actions within the permissions granted to the service account and virtual machine instance.*

### GENERATING A SERVICE ACCOUNT

To create a new service account, go to the [Service Accounts](https://console.cloud.google.com/iam-admin/serviceaccounts/serviceaccounts-zero) Page and select your project. You will be required to generate a Service account ID / email here. Go ahead and enter a name for the account and select a role for the service account, in our case, *Service Account Actor*.

You can also create a service account ID using the gcloud command.

```sh
gcloud iam service-accounts create scotch-sa --display-name "Node Service Account"
```

Once the service account is created, a JSON file will be downloaded to your local system. We will use that in the next section.

### ENVIRONMENT VARIABLES

It's always easier to set up frequently used data as environment variables in any project. Better yet, if some of these variables are sensitive, like our service account token, they are best stored away from our codebase.

To get started, let's add the project ID, cluster name, deployment name, container name and the compute zone of our cluster in *circle.yml*, these are the less sensitive details we can afford to add our configuration file.

```yaml
machine:
  environment:
    PROJECT_ID: node-app-160805
    CLUSTER_NAME: cluster-1
    COMPUTE_ZONE: us-central1-a
    #As specified in Deployment.yml
    DEPLOYMENT_NAME: nodejs-app
    CONTAINER_NAME: node-app

test:
    override:
    - mocha
```

Now to store the service account JSON file as an environment variable. It would be catastrophic if we decided to add it in the publicly accessible circle.yml file. Lucky for us, CircleCI allows us to set environment variables for each project.

First, let's encode our JSON file into base64 format so that we can store it as a string. For Linux and OS X users, run the following command to decode and copy the encoded string:

```sh
base64 scotch-2efb8709c63d.json | pbcopy
```

If you aren't able to generate locally try some online tool to convert json to base64.

Next, go to *CircleCI > Builds > Repository settings > Environment Variables > Add Variable* and add the *SERVICE_KEY* and *ACCOUNT_ID*.

We now have access to $SERVICE_KEY and $ACCOUNT_ID in our CircleCI build.

### PROVISIONING OUR DEPLOYMENT ENVIRONMENT

Just like we did in our local environment, we need to make sure that *gcloud* and *kubectl* command-line tools are installed as well as good old *Docker*. Once again, CircleCI comes pre-installed with both, we just have to make sure they are up to date before we do anything.

Once they are updated, We need to:-

+ Check for any changes in the deployment branch (master in our case)
+ Decode the service account and authenticate the build
+ Set the default project, cluster and compute zone
+ Get the credentials for the *cluster-1*
+ Start Docker

**circle.yaml**

```yaml
machine:
  environment:
    PROJECT_ID: node-app-160805
    CLUSTER_NAME: cluster-1
    COMPUTE_ZONE: us-central1-a
    #As specified in Deployment.yml
    DEPLOYMENT_NAME: nodejs-app
    CONTAINER_NAME: node-app

test:
    override:
    - mocha
#Ensure that gcloud and kubectl are updated.
dependencies:
  pre:
    - sudo /opt/google-cloud-sdk/bin/gcloud --quiet components update --version 120.0.0
    - sudo /opt/google-cloud-sdk/bin/gcloud --quiet components update --version 120.0.0 kubectl

deployment:
    production:
        branch: master
        commands:
        # Save the string to a text file
        - echo $SERVICE_KEY > key.txt
        # Decode the Service Account
        - base64 -i key.txt -d > ${HOME}/gcloud-service-key.json
        # Authenticate CircleCI with the service account file
        - sudo /opt/google-cloud-sdk/bin/gcloud auth activate-service-account ${ACCOUNT_ID} --key-file ${HOME}/gcloud-service-key.json
        # Set the default project
        - sudo /opt/google-cloud-sdk/bin/gcloud config set project $PROJECT_ID
        # Set the default container
        - sudo /opt/google-cloud-sdk/bin/gcloud --quiet config set container/cluster $CLUSTER_NAME
        # Set the compute zone
        - sudo /opt/google-cloud-sdk/bin/gcloud config set compute/zone $COMPUTE_ZONE
        # Get the cluster credentials.
        - sudo /opt/google-cloud-sdk/bin/gcloud --quiet container clusters get-credentials $CLUSTER_NAME
        # Start good old Docker
        - sudo service docker start
```

## Rolling Out Application Updates

