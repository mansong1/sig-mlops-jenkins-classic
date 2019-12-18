# MLOps with Seldon and Jenkins Classic

This repository shows how you can build a Jenkins Classic pipeline to enable Continuous Integration and Continuous Delivery (CI/CD) on your Machine Learning models leveraging Seldon for deployment.
This CI/CD pipeline will allow you to:

- Run unit tests using Jenkins Classic.
- Run end-to-end tests for your model with KIND (Kubernetes in Docker).
- Promote your model as a across multiple (staging / prod) environments.

To showcase these features we will implement add continuous integration and delivery to three different models. 
You can find these under the `/models` folder.
As we shall see, each of them will require a [different approach to deployment](#Use-Cases).

## CI/CD Pipeline

The diagram below provides a high level overview of the CI/CD pipeline.
It includes an overview of all the different types of repositories, together with the stakeholders that are the primary contributors of each, as well as the Kubernetes environments in which the applications are deployed.

The key pieces to note on the diagram are:

- There are different types of environments with different restrictions and behaviours, e.g. staging and production.
- It’s possible to have more than one environment for each type (as the type is just what would give it a specific type of config/behaviour).
- The environments are by default in the same cluster (as namespaces), however it’s also possible to configure them across different clusters.
- Each of the green boxes is a single repository, but it can also have a mono-repo approach, whereby each of the white boxes is a folder within a repo.

![CI/CD Pipeline](./images/pipeline-architecture.jpg)

**TODO:** Link each type to its own folder.

### Model implementation repository

From a high-level point of view, when a model implementation repository is updated by a Data Scientist or ML Engineer, the Jenkins CI will push changes to the [GitOps repository](#gitops-repository). This enables the following workflow:

1. A Data Scientist or ML Engineer trains a new model.
2. The Data Scientist or ML Engineer pushes the updated configuration to the model implementation repository.
3. The CI tool automatically builds and tests the model implementation.
4. The CI tool automatically pushes the change into the GitOps staging repository.
5. The CI tool automatically opens a PR into the GitOps production repository.

One key point to highlight which may not be obvious by just looking at the diagram is that in this phase of model implementation, the example above showcases how we can leverage a re-usable model server - that is, reusing a pre-built docker image instead of building one every time.
If there are more custom requirements, the user is in full control of the steps performed by the CI Platform Jenkins.
This means that it is also possible to build s2i wrapped components which may require training the image every time.

To gain a better understanding of how the CI/CD pipeline is implemented on each model implementation repository you can check the documented [deep dive](./docs/deep-dive.md).

#### Why a new repo for every model?

A new model implementation repo is currently created because it provides us with a way to separate the “Model Deployment” phase and the “Model Training/Experimentation” phase, and allows us to use the repo as the integration between any frameworks that can serve as sources of models (MLFlow, Kubeflow, Spark, etc).
The repo is able to store any metadata, IDs, and configuration files required, and is processed through the CI pipeline every time it is modified. 

#### Building a docker image in model implementation repository

Whilst most of the times users of this approach will be leveraging re-usable model servers such as the SKLearn model server, it is also possible to build a docker image every single time (i.e. build a non-reusable model every time a model changes).
This can be be done by adding the relevant steps which would most often include the s2i utility.
This may be desired if there are non-standard linux libraries or non-standard depdencies that need to be re-installed every time. 

### GitOps repository

**TODO:** All resources? Helm charts or just specs?

The state of each of our environments (e.g. production or staging) is stored on a GitOps repository.
This repository contains all the different Kubernetes resources that have been deployed to each cluster.
It is linked through ArgoCD to each of our Kubernetes clusters (or namespaces) so that a change in the repository triggers an update of our environment.

When the deployment configuration of a machine learning model implementation is updated, this will automatically make the changes available through a PR to the respective manager/tech-lead/approver.
This step will enable the end to end machine learning model promotion to be reviewed and approved by the respective individual.

The manager/tech-lead will have to approve the PR before it can be merged.
Once it’s approved, it will be merged into the GitOps repo, which will immediately trigger the update in the production namespace/cluster.

### Re-usable model server repository

If there is a need for a new reusable model server, then it’s possible to do so by creating a repository which would follow a different path.
This would be different to the model implementation repository as it would only be built once in a while, whilst the model server would be built multiple times.

## Set up

As a pre-requisite you need to ensure that have access to a Kubernetes cluster.
In particular, this guide requires the following pre- requisites:

- A Kubernetes cluster running v1.13+.
- Jenkins Classic installed in your cluster.
- Seldon Core v0.5.1 installed in your cluster.

**TODO:** Add note on ArgoCD (or Seldon Deploy??)

### Jenkins Config

The configurations required in the Jenkins server are:

- Install the GitHub Plugin [(for automated webhook triggers)](https://support.cloudbees.com/hc/en-us/articles/115003015691-GitHub-Webhook-Non-Multibranch-Jobs).
- Provide a GitHub token with read access so it can clone relevant repositories.
- Set-up webhooks so that GitHub can send push requests.

## Use cases

**TODO:** Add links to separate notebooks.

This guide goes through three different methods to build and deploy your model.

- Using Seldon pre-built re-usable model servers. 
- Using custom re-usable servers.
- Using custom servers with an embedded model.

# Diving into our CI/CD Pipeline

On this section we will dive into the internals of the CI/CD pipeline for our [model implementation repositories](README.md#model-implementation-repository).
This includes a detailed description of the `Jenkinsfile`, as well as a look into our suggested testing methodology.

Note that this will cover a generic example.
However, as we shall see, specialising this approach into any of our [three main use cases](README.md#use-cases) will be straightforward.

We leverage [Jenkins Pipelines](https://jenkins.io/doc/book/pipeline/) in order to run our continous integration and delivery automation.
From a high-level point of view, the pipeline configuration will be responsible for:

- Define a **replicable** test and build environment.
- Run the unit and integration tests (if applicable).
- Promote the application into our staging and production environments.
  
We can see a `Jenkinsfile` below taken from the [`news_classifier`](./models/news_classifier) example.
This `Jenkinsfile` defines a pipeline which takes into account all of the points mentioned above.
The following sections will dive into each of the sections in a much higher detail.


```python
!pygmentize -l groovy ../models/news_classifier/Jenkinsfile
```

    [37m//properties([pipelineTriggers([githubPush()])])[39;49;00m
    
    [36mdef[39;49;00m label = [33m"worker-${UUID.randomUUID().toString()}"[39;49;00m
    
    podTemplate(label: label, 
      workspaceVolume: dynamicPVC(requestsSize: [33m"4Gi"[39;49;00m),
      containers: [
      containerTemplate(
          name: [33m'news-classifier-builder'[39;49;00m, 
          image: [33m'seldonio/core-builder:0.4'[39;49;00m, 
          command: [33m'cat'[39;49;00m, 
          ttyEnabled: [34mtrue[39;49;00m,
          privileged: [34mtrue[39;49;00m,
          resourceRequestCpu: [33m'200m'[39;49;00m,
          resourceLimitCpu: [33m'500m'[39;49;00m,
          resourceRequestMemory: [33m'1500Mi'[39;49;00m,
          resourceLimitMemory: [33m'1500Mi'[39;49;00m,
      ),
      containerTemplate(
          name: [33m'jnlp'[39;49;00m, 
          image: [33m'jenkins/jnlp-slave:3.35-5-alpine'[39;49;00m, 
          args: [33m'${computer.jnlpmac} ${computer.name}'[39;49;00m)
    ],
    yaml:[33m'''[39;49;00m
    [33mspec:[39;49;00m
    [33m  securityContext:[39;49;00m
    [33m    fsGroup: 1000[39;49;00m
    [33m  containers:[39;49;00m
    [33m  - name: jnlp[39;49;00m
    [33m    imagePullPolicy: IfNotPresent[39;49;00m
    [33m    resources:[39;49;00m
    [33m      limits:[39;49;00m
    [33m        ephemeral-storage: "500Mi"[39;49;00m
    [33m      requests:[39;49;00m
    [33m        ephemeral-storage: "500Mi"[39;49;00m
    [33m  - name: news-classifier-builder[39;49;00m
    [33m    imagePullPolicy: IfNotPresent[39;49;00m
    [33m    resources:[39;49;00m
    [33m      limits:[39;49;00m
    [33m        ephemeral-storage: "15Gi"[39;49;00m
    [33m      requests:[39;49;00m
    [33m        ephemeral-storage: "15Gi"[39;49;00m
    [33m'''[39;49;00m,
    volumes: [
      hostPathVolume(mountPath: [33m'/sys/fs/cgroup'[39;49;00m, hostPath: [33m'/sys/fs/cgroup'[39;49;00m),
      hostPathVolume(mountPath: [33m'/lib/modules'[39;49;00m, hostPath: [33m'/lib/modules'[39;49;00m),
      emptyDirVolume(mountPath: [33m'/var/lib/docker'[39;49;00m),
    ]) {
      node(label) {
        [36mdef[39;49;00m myRepo = checkout scm
        [36mdef[39;49;00m gitCommit = myRepo.[36mGIT_COMMIT[39;49;00m
        [36mdef[39;49;00m gitBranch = myRepo.[36mGIT_BRANCH[39;49;00m
        [36mdef[39;49;00m shortGitCommit = [33m"${gitCommit[0..10]}"[39;49;00m
        [36mdef[39;49;00m previousGitCommit = sh(script: [33m"git rev-parse ${gitCommit}~"[39;49;00m, returnStdout: [34mtrue[39;49;00m)
     
        stage([33m'Test'[39;49;00m) {
          container([33m'news-classifier-builder'[39;49;00m) {
            sh [33m"""[39;49;00m
    [33m          pwd[39;49;00m
    [33m          make -C models/news_classifier \[39;49;00m
    [33m            install_dev \[39;49;00m
    [33m            test [39;49;00m
    [33m          """[39;49;00m
          }
        }
    
        [37m/* stage('Test integration') { */[39;49;00m
          [37m/* container('news-classifier-builder') { */[39;49;00m
            [37m/* sh 'models/news_classifier/integration/kind_test_all.sh' */[39;49;00m
          [37m/* } */[39;49;00m
        [37m/* } */[39;49;00m
    
        stage([33m'Promote application'[39;49;00m) {
          container([33m'news-classifier-builder'[39;49;00m) {
            withCredentials([[$class: [33m'UsernamePasswordMultiBinding'[39;49;00m,
                  credentialsId: [33m'github-access'[39;49;00m,
                  usernameVariable: [33m'GIT_USERNAME'[39;49;00m,
                  passwordVariable: [33m'GIT_PASSWORD'[39;49;00m]]) {
    
              sh [33m'models/news_classifier/promote_application.sh'[39;49;00m
            }
          }
        }
      }
    }


## Replicable test and build environment

In order to ensure that our test environments are versioned and replicable, we make use of the [Jenkins Kubernetes plugin](https://github.com/jenkinsci/kubernetes-plugin).
This will allow us to create a Docker image with all the necessary tools for testing and building our models.
Using this image, we will then spin up a separate pod, where all our build instructions will be ran.

Since it leverages Kubernetes underneath, this also ensure that our CI/CD pipelines are easily scalable.

**TODO:** Add note on `podTemplate()` object.

## Integration tests

Now that we have a model that we want to be able to deploy, we want to make sure that we run end-to-end tests on that model to make sure everything works as expected.
For this we will leverage the same framework that the Kubernetes team uses to test Kubernetes itself: [KIND](https://kind.sigs.k8s.io/).

KIND stands for Kubernetes-in-Docker, and is used to isolate a Kubernetes environent for end-to-end tests.
In our case, we will use this isolated environment to test our model.

The steps we'll have to carry out include:

1. Enable Docker within your CI/CD pod.
2. Add an integration test stage.
3. Leverage the `kind_test_all.sh` script that creates a KIND cluster and runs the tests.


### Add integration stage to Jenkins

We can leverage Jenkins Pipelines to manage the different stages of our CI/CD pipeline.
In particular, to add an integration stage, we can use the `stage()` object:

```groovy
stage('Test integration') {
  container('news-classifier-builder') {
    sh 'models/news_classifier/integration/kind_test_all.sh'
  }
}
```

### Enable Docker

To test our models, we will need to build their respective containers, for which we will need Docker.

In order to do so, we will first need to mount a few volumes into the CI/CD container.
These basically consist of the core components that docker will need to be able to run.
To mount them we will leverage the `volumes` argument of the `podTemplate()` object:

```groovy
podTemplate(...,
    volumes: [
      hostPathVolume(mountPath: '/sys/fs/cgroup', hostPath: '/sys/fs/cgroup'),
      hostPathVolume(mountPath: '/lib/modules', hostPath: '/lib/modules'),
      emptyDirVolume(mountPath: '/var/lib/docker'),
    ])
```

We then need to make sure that the pod can run with privileged context.
This step is required in order to be able to run the `docker` daemon.
To enable privileged permissions we will leverage the `privileged` flag of the `containerTemplate()` object and the `yaml` parameter of `podTemplate()`:


```groovy
podTemplate(...,
    containers: [
      containerTemplate(
          ...,
          privileged: true,
          ...
      ),
      ...],
    yaml:'''
    spec:
      securityContext:
        fsGroup: 1000
      ...
    ''',
....)
```

### Run tests in Kind 

The `kind_run_all.sh` may seem complicated at first, but it's actually quite simple. 
All the script does is set-up a kind cluster with all dependencies, deploy the model and clean everything up.
Let's break down each of the components within the script.

We first start the docker daemon and wait until Docker is running (using `docker ps q` for guidance.

```bash
# FIRST WE START THE DOCKER DAEMON
service docker start
# the service can be started but the docker socket not ready, wait for ready
WAIT_N=0
while true; do
    # docker ps -q should only work if the daemon is ready
    docker ps -q > /dev/null 2>&1 && break
    if [[ ${WAIT_N} -lt 5 ]]; then
        WAIT_N=$((WAIT_N+1))
        echo "[SETUP] Waiting for Docker to be ready, sleeping for ${WAIT_N} seconds ..."
        sleep ${WAIT_N}
    else
        echo "[SETUP] Reached maximum attempts, not waiting any longer ..."
        break
    fi
done
```

Once we're running a docker daemon, we can run the command to create our KIND cluster, and install all the components.
This will set up a Kubernetes cluster using the docker daemon (using containers as Nodes), and then install Ambassador + Seldon Core.


```bash
#######################################
# AVOID EXIT ON ERROR FOR FOLLOWING CMDS
set +o errexit

# START CLUSTER 
make kind_create_cluster
KIND_EXIT_VALUE=$?

# Ensure we reach the kubeconfig path
export KUBECONFIG=$(kind get kubeconfig-path)

# ONLY RUN THE FOLLOWING IF SUCCESS
if [[ ${KIND_EXIT_VALUE} -eq 0 ]]; then
    # KIND CLUSTER SETUP
    make kind_setup
    SETUP_EXIT_VALUE=$?
```

We can now run the tests; for this we run all the dev installations and kick off our tests (which we'll add inside of the integration folder).

```bash
    # BUILD S2I BASE IMAGES
    make build
    S2I_EXIT_VALUE=$?

    ## INSTALL ALL REQUIRED DEPENDENCIES
    make install_integration_dev
    INSTALL_EXIT_VALUE=$?
    
    ## RUNNING TESTS AND CAPTURING ERROR
    make test
    TEST_EXIT_VALUE=$?
fi
```


Finally we just clean everything, including the cluster, the containers and the docker daemon.

```bash
# DELETE KIND CLUSTER
make kind_delete_cluster
DELETE_EXIT_VALUE=$?

#######################################
# EXIT STOPS COMMANDS FROM HERE ONWARDS
set -o errexit

# CLEANING DOCKER
docker ps -aq | xargs -r docker rm -f || true
service docker stop || true
```

## Promote your application

After running our integration tests, the last step is to promote our model to our staging and production environments.
For that, we will leverage our [GitOps repository](./README.md#gitops-repository) where the state of each environment is stored.

In particular, we will:

- Push a change to the staging GitOps repository, which will update the staging environment instantly.
- Submit a PR to the production GitOps repository, which will wait for a Tech Lead / Manager approval.

This will be handled by the `promote_application.sh` script, which can be seen below.


```python
!pygmentize ../models/news_classifier/promote_application.sh
```

    [37m#!/bin/bash[39;49;00m
    
    [37m# ENSURE WE ARE IN THE DIR OF SCRIPT[39;49;00m
    [36mcd[39;49;00m -P -- [33m"[39;49;00m[34m$([39;49;00mdirname -- [33m"[39;49;00m[31m$0[39;49;00m[33m"[39;49;00m[34m)[39;49;00m[33m"[39;49;00m
    [37m# SO WE CAN MOVE RELATIVE TO THE ACTUAL BASE DIR[39;49;00m
    
    [36mexport[39;49;00m [31mGITOPS_REPO[39;49;00m=[33m"seldon-gitops"[39;49;00m
    [36mexport[39;49;00m [31mGITOPS_ORG[39;49;00m=[33m"adriangonz"[39;49;00m
    [36mexport[39;49;00m [31mSTAGING_FOLDER[39;49;00m=[33m"staging"[39;49;00m
    [36mexport[39;49;00m [31mPROD_FOLDER[39;49;00m=[33m"production"[39;49;00m
    
    [37m# This is the user that is going to be assigned to PRs[39;49;00m
    [36mexport[39;49;00m [31mGIT_MANAGER[39;49;00m=[33m"adriangonz"[39;49;00m
    
    [36mexport[39;49;00m [31mUUID[39;49;00m=[34m$([39;49;00mcat /proc/sys/kernel/random/uuid[34m)[39;49;00m
    
    git clone https://[33m${[39;49;00m[31mGIT_USERNAME[39;49;00m[33m}[39;49;00m:[33m${[39;49;00m[31mGIT_PASSWORD[39;49;00m[33m}[39;49;00m@github.com/[33m${[39;49;00m[31mGITOPS_ORG[39;49;00m[33m}[39;49;00m/[33m${[39;49;00m[31mGITOPS_REPO[39;49;00m[33m}[39;49;00m
    
    [36mcd[39;49;00m [33m${[39;49;00m[31mGITOPS_REPO[39;49;00m[33m}[39;49;00m
    cp -r ../charts/* [33m${[39;49;00m[31mSTAGING_FOLDER[39;49;00m[33m}[39;49;00m/.
    ls [33m${[39;49;00m[31mSTAGING_FOLDER[39;49;00m[33m}[39;49;00m
    
    [37m# Check if any modifications identified[39;49;00m
    git add -N [33m${[39;49;00m[31mSTAGING_FOLDER[39;49;00m[33m}[39;49;00m/
    git --no-pager diff --exit-code --name-only origin/master [33m${[39;49;00m[31mSTAGING_FOLDER[39;49;00m[33m}[39;49;00m
    [31mSTAGING_MODIFIED[39;49;00m=[31m$?[39;49;00m
    [34mif[39;49;00m [[ [31m$STAGING_MODIFIED[39;49;00m -eq [34m0[39;49;00m ]]; [34mthen[39;49;00m
      [36mecho[39;49;00m [33m"Staging env not modified"[39;49;00m
      [36mexit[39;49;00m [34m0[39;49;00m
    [34mfi[39;49;00m
    
    [37m# Adding changes to staging repo automatically[39;49;00m
    git add [33m${[39;49;00m[31mSTAGING_FOLDER[39;49;00m[33m}[39;49;00m/
    git commit -m [33m'{"Action":"Deployment created","Message":"","Author":"","Email":""}'[39;49;00m
    git push https://[33m${[39;49;00m[31mGIT_USERNAME[39;49;00m[33m}[39;49;00m:[33m${[39;49;00m[31mGIT_PASSWORD[39;49;00m[33m}[39;49;00m@github.com/[33m${[39;49;00m[31mGITOPS_ORG[39;49;00m[33m}[39;49;00m/[33m${[39;49;00m[31mGITOPS_REPO[39;49;00m[33m}[39;49;00m
    
    [37m# Add PR to prod[39;49;00m
    cp -r ../charts/* production/.
    
    [37m# Create branch and push[39;49;00m
    git checkout -b [33m${[39;49;00m[31mUUID[39;49;00m[33m}[39;49;00m
    git add [33m${[39;49;00m[31mPROD_FOLDER[39;49;00m[33m}[39;49;00m/
    git commit -m [33m'{"Action":"Moving deployment to production repo","Message":"","Author":"","Email":""}'[39;49;00m
    git push https://[33m${[39;49;00m[31mGIT_USERNAME[39;49;00m[33m}[39;49;00m:[33m${[39;49;00m[31mGIT_PASSWORD[39;49;00m[33m}[39;49;00m@github.com/[33m${[39;49;00m[31mGITOPS_ORG[39;49;00m[33m}[39;49;00m/[33m${[39;49;00m[31mGITOPS_REPO[39;49;00m[33m}[39;49;00m [33m${[39;49;00m[31mUUID[39;49;00m[33m}[39;49;00m
    
    [37m# Create pull request[39;49;00m
    [36mexport[39;49;00m [31mPR_RESULT[39;49;00m=[34m$([39;49;00mcurl [33m\[39;49;00m
      -u [33m${[39;49;00m[31mGIT_USERNAME[39;49;00m[33m}[39;49;00m:[33m${[39;49;00m[31mGIT_PASSWORD[39;49;00m[33m}[39;49;00m [33m\[39;49;00m
      -v -H [33m"Content-Type: application/json"[39;49;00m [33m\[39;49;00m
      -X POST -d [33m"[39;49;00m[33m{\"title\": \"SeldonDeployment Model Promotion Request - UUID: [39;49;00m[33m${[39;49;00m[31mUUID[39;49;00m[33m}[39;49;00m[33m\", \"body\": \"This PR contains the deployment for the Seldon Deploy model and has been allocated for review and approval for relevant manager.\", \"head\": \"[39;49;00m[33m${[39;49;00m[31mUUID[39;49;00m[33m}[39;49;00m[33m\", \"base\": \"master\" }[39;49;00m[33m"[39;49;00m [33m\[39;49;00m
      https://api.github.com/repos/[31m$GITOPS_ORG[39;49;00m/[31m$GITOPS_REPO[39;49;00m/pulls[34m)[39;49;00m
    [36mexport[39;49;00m [31mISSUE_NUMBER[39;49;00m=[34m$([39;49;00m[36mecho[39;49;00m [33m\[39;49;00m
      [31m$PR_RESULT[39;49;00m |
      python -c [33m'import json,sys;obj=json.load(sys.stdin);print(obj["number"])'[39;49;00m[34m)[39;49;00m
    
    [37m# Assign PR to relevant user[39;49;00m
    curl [33m\[39;49;00m
      -u [33m${[39;49;00m[31mGIT_USERNAME[39;49;00m[33m}[39;49;00m:[33m${[39;49;00m[31mGIT_PASSWORD[39;49;00m[33m}[39;49;00m [33m\[39;49;00m
      -v -H [33m"Content-Type: application/json"[39;49;00m [33m\[39;49;00m
      -X POST -d [33m"[39;49;00m[33m{\"assignees\": [\"[39;49;00m[33m${[39;49;00m[31mGIT_MANAGER[39;49;00m[33m}[39;49;00m[33m\"] }[39;49;00m[33m"[39;49;00m [33m\[39;49;00m
      https://api.github.com/repos/[31m$GITOPS_ORG[39;49;00m/[31m$GITOPS_REPO[39;49;00m/issues/[31m$ISSUE_NUMBER[39;49;00m



```python

```
