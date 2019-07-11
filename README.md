# Setup of Full SPLAT Environment and Deployment

### Kubernetes & Helm
--

_Kubernetes is the platform running the saervices. It's a really big subject in itself, and while I recommend learning it as much as possible, it's not necessary to do deployments or follow this guide_

_Helm is an application that allows you to treat all your Kubernetes files as a group. It also adds the capability for variables and scripting, which provide a lot of flexibility for deployment strategies._

_This document does not cover Kubernetes or Helm in detail, there are other materials which already do that. You should be able to perform all the commands shown here regardless._

#### Installing Helm

_This step is only necessary if you will be modifying the Chart.yaml file_.

For instructions for your operating system see [this documentation](https://helm.sh/docs/using_helm/).
#### Charts and other files

Helm requires a file `Chart.yaml` with some basic options.

```
appVersion: 1.0.0
description: Helm chart for Splat microservices deployment
maintainers:
- name: NCR, TinRoof Software
name: splat-deployer
version: 0.9.1
```

The most important items to notice are `name` and `version`. Those will be needed later to set up Harness.

Alongside the `Chart.yaml` file, there should be a `templates` directory. Optionally, there should be a `values.yaml` file as well.

```
Chart.yaml
templates
 |_a_k8s_template.yaml
 |_another_k8s_template.yaml
values.yaml
```

In the `templates` directory we have files for [services](https://kubernetes.io/docs/concepts/services-networking/service/) and [deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) . These are Kubernetes terms you may want to familiarize yourself with.

Below are the service and deployment definition files for the Splat Orchestrator. These are for example pusposes only, and you will see more files [insert repo here]

`orchestrator-deployment-definition.yaml`

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orchestrator-deployment
  labels:
    app: splat
spec:
  replicas: {{ .Values.deploymentReplicas }}
  selector:
    matchLabels:
      name: orchestrator-pod
      app: splat
  template:
    metadata:
      name: orchestrator-pod
      labels:
        name: orchestrator-pod
        app: splat
    spec:
      containers:
      - name: splat-orchestrator
        image: {{ .Values.orchestratorImage }}
        ports:
          - containerPort: {{ .Values.orchestratorContainerPort }}
```

`orchestrator-service-definition`

```
apiVersion: v1
kind: Service
metadata:
  name: orchestrator-service
  labels:
    name: orchestrator-service
    app: splat
spec:
  type: NodePort
  selector:
    name: orchestrator-pod
    app: splat
  ports:
  - port: 80

```

You will notice the `{{ .Values.xxxxxxxx }}` syntax in the file above. These are replaced with values from the `values.yaml` file.

`values.yaml`

```
deploymentReplicas: 3
orchestratorImage: 'us.gcr.io/hsp-splat-dev/splat-orchestrator:latest'
orchestratorContainerPort: 80
connectorImage: 'us.gcr.io/hsp-splat-dev/splat-connector:latest'
connectorContainerPort: 80
```

#### Packaging the Helm bundle

In the directory where `Chart.yaml` is, run the follwoing command.

```
$ helm package .
Successfully packaged chart and saved it to: /Users/nicrosental/projects/SPLAT/helm-charts/gke/splat-deployer/splat-deployer-0.9.1.tgz
```

It's very likely you see an error `Error: directory name (xxxxxxxxx) and Chart.yaml name (yyyyyyy) must match`. This is a known Helm quirk. The `name` value in the `Chart.yaml` has to match the name of the directory where it's contained. In our case the directory should be `splat-deployer`.

#### Publishing the Helm bundle

We use [Google Cloud Storage (GCS)](https://console.cloud.google.com/storage/browser?project=hsp-splat-dev&folder=&organizationId=1052607630679&angularJsUrl=%2Fstorage%2Fbrowser%3Fproject%3Dhsp-splat-dev%26folder%26organizationId%3D1052607630679&authuser=3) to host the Helm bundle. 
Choose a directory structure, and upload the `.tgz` file. You will need this location to set up Harness.

![gcs-helm](https://cl.ly/8175a1661930/Screen%20Shot%202019-07-10%20at%209.41.15%20AM.png)

At this point the Helm bundle is ready for the Helm application to perform deployments. We'll return to the Helm bundle during the Harness setup section.

### Setting up the cluster
--

Access the [Google Kubernetes Engine](https://console.cloud.google.com/kubernetes/list?project=hsp-splat-dev&folder&organizationId=1052607630679) in the Google Cloud Console. Ensure `hsp-splat-dev` has been selected as the project in the top left pull down.

![select project](https://cl.ly/b989b8bbf070/[a03ee2b9fd5d353fb733bc14284d75a1]_Screen%20Shot%202019-07-09%20at%204.17.12%20PM.png)

Select `Clusters` then `Create Cluster`.

![create cluster](https://cl.ly/fbbedabf78d9/[057ce0cd9153ff2db5133531f94a3e0c]_Screen%2520Shot%25202019-07-09%2520at%25204.21.17%2520PM.png)

Fill out a name for the cluster. The defaults for the cluster type, and the zone should be okay, but change them if desired. Down in the `default pool` area, increase the machine type to `4 CPU`. Feel free to play around with the number of nodes, and CPUs, but check the cost calculator. The combination of 3 nodes, with 4 CPU (n1-standard-4) works well.

![4cpu](https://cl.ly/8ad602f41d65/Screen%20Shot%202019-07-09%20at%204.23.46%20PM.png)

#### Connecting to the new cluster
After a few minutes the new cluster should appear with a green checkmark next to it in the clusters list. Click on the corresponding `Connect` button.

Copy the command displayed in the dialog.

![connect to cluster](https://cl.ly/c20dbc7e3e96/Screen%20Shot%202019-07-09%20at%204.40.13%20PM.png)

Paste the command in your computer's command line.

_This step assumes `gcloud` to be installed locally. For installation instructions see [this document](https://cloud.google.com/sdk/install)._

```
$ gcloud container clusters get-credentials splat-prerelease-cluster --zone us-central1-a --project hsp-splat-dev

Fetching cluster endpoint and auth data.
kubeconfig entry generated for splat-prerelease-cluster.
```

### Deploying the Harness delegate
--
_The Harness delegate is deployed to the cluster, and it acts as the link between GKE and Harness. For more information regarding the Harness delegate see [this article](https://docs.harness.io/article/h9tkwmkrm7-delegate-installation)_ 

Log on to the [Harness dashboard](http://harness.ncrcloud.com/) (only works from the NCR network)

Select `Setup`, then `Harness Delegates`.

![delegates](https://cl.ly/794925899f09/[de6d27b04df9fa295f07c1f8538c45b9]_Screen%2520Shot%25202019-07-09%2520at%25204.31.56%2520PM.png)

Select `Download Delegate`, then `Kubernetes YAML`. Give the delegate a recognizable name (something related to the cluster's name is preferable.) Select `gke-cluster-delegate-profile` in the `Profile` field.

![delegate setup](https://cl.ly/08ddd43fa080/Screen%20Shot%202019-07-09%20at%204.49.38%20PM.png)

Save the file, and note its location. Open said location at the command line. Decompress the file, and `cd` into the new directory.

```
$ tar -xf harness-delegate-kubernetes\(8\).tar.gz
$ cd harness-delegate-kubernetes
$ kubectl apply -f harness-delegate.yaml

namespace/harness-delegate created
clusterrolebinding.rbac.authorization.k8s.io/harness-delegate-cluster-admin created
secret/splat-prerelease-delegate-proxy created
statefulset.apps/splat-prerelease-delegate-jkrddg created
```

This should deploy the delegate to the cluster. The delegate should now be visible in Harness, and in the GKE dashboard. 

![in harness](https://cl.ly/c2b67ef72d71/[3dd316dc17492ceddd35868523504ecb]_Screen%2520Shot%25202019-07-09%2520at%25204.57.24%2520PM.png)

To verify that it's running, run the following command.

```
$ kubectl get pods --namespace=harness-delegate
NAME                                 READY   STATUS    RESTARTS   AGE
splat-prerelease-delegate-jkrddg-0   1/1     Running   0          3m42s
```

### Pipeline Assembly
--
Getting the pipeline ready involves multiple steps within Harness. We'll break down each one, and present them in the desired order.

#### Set up the cluster as a Cloud Provider

Select `Setup` > `Cloud Providers` > `Add Cloud Provider`

Fill out the form as follows, ensuring the name provided is descriptive of the cluster. Select the appropriate delegate, and click `Test` to ensure it's been set up properly. Submit it.

![cloud provider](https://cl.ly/c28fbd53112d/Screen%20Shot%202019-07-09%20at%205.16.53%20PM.png)

#### Set up the Helm bundle as an artifact server
Select `Setup` > `Connectors` > `Artifact Servers` > `Add Artifact Server`

Fill out the form as follows. The bucket name must match the one selected on GCS.

![artifact server](https://cl.ly/fce023a23538/Screen%20Shot%202019-07-10%20at%2011.13.24%20AM.png)

#### Set up the service
Select `Setup` > `Splat` > `Service` > `Add Service`

Fill the form out as follows, use a descriptive name.

![helm service](https://cl.ly/57c1ad3d7542/Screen%20Shot%202019-07-09%20at%205.22.03%20PM.png)

After submitting, you will be redirected to a new screen. Select `Add Chart Specification`. Fill out this form, selecting the Helm repository set up in the previous section.

![service set up](https://cl.ly/757c13917c64/Screen%20Shot%202019-07-10%20at%202.48.48%20PM.png)

#### Set up the environment
Select `Setup` > `Splat` > `Environments` > `Add Environment`

Fill out the form, and submit it.

![environment](https://cl.ly/600ee6a55979/Screen%20Shot%202019-07-09%20at%205.08.31%20PM.png)

The new environment will be displayed on the list. It will very likely be labeled `Setup Required`. Click on the title, then `Add Service Infrastructure`.

Select the service, and the cloud provider that were set up in the previous steps.

![service infrastructure](https://cl.ly/1413193b7816/Screen%20Shot%202019-07-09%20at%205.24.22%20PM.png)

#### Set up workflows

_Workflows are actions that can be chained together to form a pipeline. A workflow may package a Docker image, push code, etc. They can also run individually, outside of a pipeline._ 

The first workflow we set up is to push the container to the Kubernetes cluster. Select `Setup` > `Splat` > `Workflows` > `Add workflow`. Fill out the form as follows.


![env workflow](https://cl.ly/97bcec236d67/Screen%20Shot%202019-07-09%20at%205.32.51%20PM.png)


Upon submission, the `Workflow Overview` screen will be displayed.

![workflow overview](https://cl.ly/87d91c33c2f9/Screen%20Shot%202019-07-09%20at%205.35.24%20PM.png)

Select `Helm Deploy (Incomplete)` to finish the set up.

Fill out the form as follows.

![helm deploy](https://cl.ly/cfe2c5bca623/Screen%20Shot%202019-07-09%20at%205.38.22%20PM.png)

Next, set up a Cloudbuild workflow.

_Cloudbuild is a Google Cloud service that allows building Docker images based on a configuration file, and pushed them to Google Cloud Registry. The configuration file is `cloudbuild.yaml` and it is included in the root of the source code for each Splat microservice._

```
# cloudbuild.yaml
steps:
- name: 'gcr.io/cloud-builders/yarn'
  args: ['install']
- name: 'gcr.io/cloud-builders/docker'
  args: ['build', '--tag=us.gcr.io/hsp-splat-dev/splat-connector', '.']
- name: 'us.gcr.io/odsp-management/ncr-cloud-builders-helm'
  env:
  - 'HELM_REPO_NAME=splat-helm'
  - 'HELM_REPO_URL=gs://splat-helm-chart'
  - 'CHART_PATH=helm'
  - 'CHART_VERSION=0.8.1'
  - 'CHART_NAME=splat-deployer'
  - 'APP_VERSION=$REVISION_ID'
images: ['us.gcr.io/hsp-splat-dev/splat-connector']
options:
  workerPool: 'odsp-management/cloud-build-workers'
```

Select `Setup` > `Splat` > `Workflows` > `Add workflow`. Fill out the form as follows.

![cloudbuild form](https://cl.ly/15514b25a294/Screen%20Shot%202019-07-10%20at%203.06.23%20PM.png)

Upon submission, the `Workflow Overview` screen will be displayed.

![workflow overview](https://cl.ly/87d91c33c2f9/Screen%20Shot%202019-07-09%20at%205.35.24%20PM.png)

Select  the `X` next to `Artifact Collection (Incomplete)` to delete it. Then select `Add Command` in the `Collect Artifact` section.

Select `Shell Script` from the dialog, then fill out the form as follows (actual code below screenshot.) Make sure that it's pointing to the right repository.

![shell script](https://cl.ly/83d16e125c7f/Screen%20Shot%202019-07-10%20at%203.13.56%20PM.png)

```
echo "BUILDING SPLAT"

echo ${BUILD.APP_VERSION}

# REMOVE PREVIOUS SPLAT DIR
rm -rf splat

# CLONE THE REPOSITORY
git clone https://almgit.ncr.com/scm/nolo/splat.git

# CD INTO THE DIRECTORY
cd splat

# SUBMIT CLOUD BUILD
gcloud builds submit --config=cloudbuild.yaml
```

At this point all that's left to do before deployment is putting together a pipeline.

#### Set up pipeline

_There can (and should) be multiple pipelines for different purposes. This example is a very simple pipeline that builds the image on Cloudbuild, then pushes it to the Kubernetes cluster._

Select `Setup` > `Splat` > `Pipelines` > `Add Pipeline`

Fill out the name, and description, then submit.

Select `Click to add Pipeline Stage`. Fill out the form, selecting the Cloudbuild workflow created in the previos step.

![pipeline workflow](https://cl.ly/34d073e7b0b0/Screen%20Shot%202019-07-10%20at%203.24.24%20PM.png)

Repeat the sequence to add a Workflow, this time selecting the push Workflow created in the previous step.

The pipeline should be complete, select `Deploy`, and watch it run.

![deploy](https://cl.ly/aaaeae438c39/Screen%20Shot%202019-07-10%20at%203.26.44%20PM.png)

#### Set up triggers

At this point you have a pipeline that can make deployments. We will set jup triggers to kick it off when code gets merged.

Select `Setup` > `Splat` > `Triggers` > `Add Trigger`

_Webhook/trigger interaction between Harness and Bitbucket is really finicky. The combination shown here will allow you to link distinct pipelines to specific branches._

Fill out the form as follows.

![trigger](https://cl.ly/2ca8bd6b7fd5/Screen%252520Shot%2525202019-07-11%252520at%25252011.00.56%252520AM.png)

_The *repo:push* event works when pushing code directly to, or when merging code into the matching branch. Merging code into a branch with the prefix `prerelease/` will kick off the pipeline._ 

Once created, find the trigger in the list and select `Bitbucket Webhook`, and copy the displayed URL.

Head over to your Bitbucket repo. 

Select `Settings` > `Post Webhooks` > `Add webhook`

Fill out the form as follows, pasting the Harness trigger URL in the URL field.

![bitbucket webhook](https://cl.ly/a0e4dd7319ac/Screen%252520Shot%2525202019-07-11%252520at%25252011.08.49%252520AM.png)

# AWESOMESAUCE, YOU MADE IT TO THE END 🙌

## Upcoming subjects:
* Docker images
* Approval pipelines.
* GKE Dashboard




