# Using WebLogic on OpenShift
## Introduction
This document is a guide to getting started with the [WebLogic Kubernetes Operator](https://github.com/oracle/weblogic-kubernetes-operator) on OpenShift using the [domain in image source type](https://oracle.github.io/weblogic-kubernetes-operator/userguide/managing-domains/choosing-a-model/). This guide is based on the [quick start provided in the operator documentation](https://oracle.github.io/weblogic-kubernetes-operator/quickstart/) and [this blog post written by Mark Nelson](https://blogs.oracle.com/weblogicserver/running-weblogic-on-openshift). 

## Prerequisites
- You must have an account that has permission to access the Oracle Container Registry. This enables you to pull the base image used in the steps below. If you do not already have an account, you can go to https://container-registry.oracle.com and create one.
- You must have the OpenShift command line tools installed (including `oc` and `kubectl`). You can download the latest tools from [this link](https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/stable-4.7/) (you'll look for the `openshift-client-{{ operating-system }}` file). 
- You must have Helm (v3) installed on your local machine. You can download the latest tools from [this link](https://mirror.openshift.com/pub/openshift-v4/clients/helm/latest/) (you'll look for the `helm-{{ operating-system }}-{{ architecture }}` file).
- You must have either Docker or Podman installed locally on your machine. We will be used it to pull, push, and build images. If you are looking to use Docker, follow these docs to install on [Mac](https://docs.docker.com/docker-for-mac/install/) or [Windows](https://docs.docker.com/docker-for-windows/install/)). If you're looking to use Podman (Linux) ensure that you have an alias set up for the docker command to route to podman (`alias docker=podman`).
- You must have `cluster-admin` access to your OpenShift cluster. 

### Step 1

On your local machine, clone this repository by running the following command:

```
git clone --recurse-submodules https://github.com/Red-Hat-SE-RTO/weblogic-operator-on-openshift.git
```

Then, change directory to the newly cloned repository by running the following command:
```
cd weblogic-operator-on-openshift/
```

### Step 2

Login to the Oracle Container Registry to allow Docker to pull images using your credentials by running the following command:

```
docker login container-registry.oracle.com
```

### Step 3 

Login to OpenShift using your credentials. To get a token to login, follow these directions:

1. In the OpenShift Web Console, click on your name in the top right hand corner and select "Copy Login Command". 
2. Once you authenticate, select "Display Token"
3. Copy the `oc` command displayed. 
4. Run the command in your terminal. 
5. Do NOT close this window out, as we will need it in Step 5.

The command will be similar to:
```
oc login --token=sha256~xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx --server=https://api.ocp.example.com:6443
```

The output will be similar to:

```
Logged into "https://api.ocp.example.com:6443" as "johndoe@example.com" using the token provided.

You have access to 70 projects, the list has been suppressed. You can list all projects with ' projects'

Using project "default".
```

### Step 4 

Ensure that the OpenShift registry is exposed by running the following command:

```
oc patch configs.imageregistry.operator.openshift.io/cluster --patch '{"spec":{"defaultRoute":true}}' --type=merge
```

### Step 5

Login to the OpenShift registry to allow Docker to push images using your credentials by running the following command:

```
docker login $(oc get route default-route -n openshift-image-registry --template='{{ .spec.host }}')
```

When prompted, enter your OpenShift username, and the token from Step 3 above. (The token will look like `sha256~xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`, do not enter the `oc` command).


### Step 6

Pull the base image for the domain from the Oracle image registry by running the following command:
```
docker pull container-registry.oracle.com/middleware/weblogic:12.2.1.4
```

The output will be similar to:
```
12.2.1.4: Pulling from middleware/weblogic
401a42e1eb4f: Pull complete
5779b03f4f45: Pull complete
1ea9ed498323: Pull complete
b99f19d3cc6a: Pull complete
3d288a26d69b: Pull complete
a1a80dd8562a: Pull complete
Digest: sha256:16eccb81a4ccf146326bad6bd9a74fb259799f5d968c6714aea80521197ae528
Status: Downloaded newer image for container-registry.oracle.com/middleware/weblogic:12.2.1.4
container-registry.oracle.com/middleware/weblogic:12.2.1.4
```

### Step 7

Create a project (namespace) to install the WebLogic Kubernetes Operator into by running the following command:

```
oc new-project weblogic-operator
```

### Step 8

Create a service account for the operator to use by running the following command:

```
oc create serviceaccount -n weblogic-operator weblogic-operator-sa
```

### Step 9

Install the operator using Helm by running the following command:
```
helm install weblogic-operator weblogic-kubernetes-operator/kubernetes/charts/weblogic-operator \
  --namespace weblogic-operator \
  --set image=ghcr.io/oracle/weblogic-kubernetes-operator:3.2.2 \
  --set serviceAccount=weblogic-operator-sa \
  --set "enableClusterRoleBinding=true" \
  --set "domainNamespaceSelectionStrategy=LabelSelector" \
  --set "domainNamespaceLabelSelector=weblogic-operator\=enabled" \
  --wait
```

If successful, the output will be similar to:
```
NAME: weblogic-operator
LAST DEPLOYED: Wed May 19 11:30:49 2021
NAMESPACE: weblogic-operator
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

The Helm command we ran configured the operator to manage domains in any OpenShift project (namespace) with the label, “weblogic-operator=enabled”. 

### Step 10

Validate the operator deployment by running the following command:
```
oc get pods -n weblogic-operator
```

If successful, the output will be similar to:
```
NAME                                 READY   STATUS    RESTARTS   AGE
weblogic-operator-6bb58697b7-2zbql   1/1     Running   0          87s
```

Ensure that the pod is "Running" and is "Ready (1/1)".

If you look at the pod logs, you may see the error message:
```
Operator cannot proceed, as the Custom Resource Definition for ''domains.weblogic.oracle'' is not installed.
```
However, once there are domains for the operator to manage, this message should disappear.

### Step 11

Create a project (namespace) to deploy our domain to by running the following command:
```
oc new-project sample-domain1
```

### Step 12

Label the project (namespace) to ensure the operator knows to manage it.
```
oc label ns sample-domain1 weblogic-operator=enabled
```

### Step 13

The operator expects a Kubernetes secret to exist with the credentials for the WebLogic administrator. The password **must** have at least 8 alphanumeric characters with at least one number or special character. If you do not follow this requirement, the domain creation will fail.

To create a secret using the default credentials, run the following command:
```
./weblogic-kubernetes-operator/kubernetes/samples/scripts/create-weblogic-domain-credentials/create-weblogic-credentials.sh \
-u administrator -p AbCdEfG123! -n sample-domain1 -d domain1
```

The output will be similar to:
```
secret/domain1-weblogic-credentials created
secret/domain1-weblogic-credentials labeled
The secret domain1-weblogic-credentials has been successfully created in the sample-domain1 namespace.
```

> Important note: if you change the username and password (as you should in your enterprise environment) in the above commands, you will need to also change them in the `properties/docker-build/adminpass.properties`, `properties/docker-build/adminuser.properties`, and `properties/docker-run/security.properties` files. 

### Step 14

Download the WebLogic Deploy Tooling to your local working directory. You can download the latest release directly from Oracle [here](https://github.com/oracle/weblogic-deploy-tooling/releases/latest).

You can download version 1.9.12 using the following command:

```
wget https://github.com/oracle/weblogic-deploy-tooling/releases/download/release-1.9.12/weblogic-deploy.zip
```

### Step 15

Using the `build-archive.sh` script, build the sample application we will be deploying using the following command:

```
./build-archive.sh
```

The output will be similar to:
```
[INFO] Installing /Users/mmcneill/Git/weblogic-on-openshift/test-webapp/target/testwebapp.war to /Users/mmcneill/.m2/repository/com/oracle/weblogic/testwebapp/1.0/testwebapp-1.0.war
[INFO] Installing /Users/mmcneill/Git/weblogic-on-openshift/test-webapp/pom.xml to /Users/mmcneill/.m2/repository/com/oracle/weblogic/testwebapp/1.0/testwebapp-1.0.pom
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  14.920 s
[INFO] Finished at: 2021-05-24T10:01:37-04:00
[INFO] ------------------------------------------------------------------------
added manifest
adding: wlsdeploy/(in = 0) (out= 0)(stored 0%)
adding: wlsdeploy/applications/(in = 0) (out= 0)(stored 0%)
adding: wlsdeploy/applications/testwebapp.war(in = 3548) (out= 2507)(deflated 29%)
```

### Step 16

Using the `quickBuild.sh` script, build the container image that contains our custom application using the following command:

```
./quickBuild.sh
```

The output will be similar to:
```
 => exporting to image                                                                                                                                                                                                                   8.9s
 => => exporting layers                                                                                                                                                                                                                  8.9s
 => => writing image sha256:68c20783949fa57a3dffae491f3f68510c509cf31eea30de9dbdc31857ae65f5                                                                                                                                             0.0s
 => => naming to docker.io/library/my-domain1-image:1.0
```

### Step 19

Tag and push our newly created image to the OpenShift registry by running the following commands:

```
docker tag my-domain1-image:1.0 $(oc get route default-route -n openshift-image-registry --template='{{ .spec.host }}')/sample-domain1/my-domain1-image:1.0
docker push $(oc get route default-route -n openshift-image-registry --template='{{ .spec.host }}')/sample-domain1/my-domain1-image:1.0
```

The output will be similar to:
```
5f70bf18a086: Pushed
5126ea77cec1: Pushed
f9189f4c273f: Pushed
bc683693f6e2: Pushed
ad02d087cd2b: Pushed
67a5c7e7416c: Pushed
1d00dd0976a4: Pushed
cdcca75bb742: Pushed
d2d80721548e: Pushed
f44d1cb58cca: Pushed
9b4be6c23054: Pushed
3f0e18db1c65: Pushed
32eeb31c2f24: Pushed
1.0: digest: sha256:5044fc62fd72918d75c40c2363738897d3b8f5143109e0e51ecc38e56d6f9e4d size: 3253
```

### Step 18

Create the WebLogic Domain Custom Resource (CR) object in OpenShift by running the following command:

```
oc apply -f sample-domain.yaml
```

### Step 19

We now need to expose both the admin server and the application frontend, using OpenShift's built-in ingress controller. This will enable us to access the admin console, use tooling like WLST, and access our newly deployed WebLogic application. To expose the operator-created services, by running the following command:

```
oc expose service domain1-admin-server-ext --port=default
oc expose service domain1-cluster-cluster-1 --port=default
```

### Step 20

You are now ready to access the admin console or the application in your web browser. 

To get the host for the admin console, run the following command: 
```
oc get route domain1-admin-server-ext -n sample-domain1 --template='{{ .spec.host }}'
```

Once you have the host, going to `http://{{ host }}/console` will allow you to authenticate with the credentials created in step 13 above.

To get the host for the WebLogic application, run the following command: 
```
oc get route domain1-cluster-cluster-1 -n sample-domain1 --template='{{ .spec.host }}'
```

Once you have the host, going to `http://{{ host }}/testwebapp` will show you our test application that was deployed to WebLogic.