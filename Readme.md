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

## Adding Tests

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

## INTEGRATING CIRCLECI

With our tests passing, let's add a *circle.yml* file and instruct CircleCI how to run our tests using mocha. Luckily, every CircleCI build comes preinstalled with mocha so all we need to do is instruct it to use mocha.

Note that by default, CircleCI will try to run *npm test*.

```yaml
test:
    override:
    - mocha
```


## ADD PROJECT TO CIRCLECI

Once everything is set up, push the code to a Gihub repository, login to CircleCI with Github then go to CircleCI > Add Projects and build your repository. If everything is correctly set up, you should have a succesful build.
