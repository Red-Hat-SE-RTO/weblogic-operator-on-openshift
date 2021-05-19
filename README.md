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

### Step 5

Create a project (namespace) to install the WebLogic Kubernetes Operator into by running the following command:

```
oc new-project weblogic-operator
```

### Step 6

Create a service account for the operator to use by running the following command:

```
oc create serviceaccount -n weblogic-operator weblogic-operator-sa
```

### Step 7

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

### Step 8

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

### Step 9

Create a project (namespace) to deploy our domain to by running the following command:
```
oc new-project sample-domain1
```

### Step 10

Label the project (namespace) to ensure the operator knows to manage it.
```
oc label ns sample-domain1 weblogic-operator=enabled
```

### Step 11

The operator expects a Kubernetes secret to exist with the credentials for the WebLogic administrator. The password **must** have at least 8 alphanumeric characters with at least one number or special character. If you do not follow this requirement, the domain creation will fail.

Run the following command, making sure you replace the content within the `{{}}` with the content specified.

```
./weblogic-kubernetes-operator/kubernetes/samples/scripts/create-weblogic-domain-credentials/create-weblogic-credentials.sh \
-u {{ username }} -p {{ password }} -n sample-domain1 -d sample-domain1
```

For example, the command will be similar to:
```
./weblogic-kubernetes-operator/kubernetes/samples/scripts/create-weblogic-domain-credentials/create-weblogic-credentials.sh \
-u administrator -p AbCdEfG123! -n sample-domain1 -d sample-domain1
```

The output will be similar to:
```
secret/sample-domain1-weblogic-credentials created
secret/sample-domain1-weblogic-credentials labeled
The secret sample-domain1-weblogic-credentials has been successfully created in the sample-domain1 namespace.
```

### Step 12
