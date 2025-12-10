> This repository is a fork of [openshift-for-developers/hello](https://github.com/openshift-for-developers/hello), adapted to document and demonstrate my solution to the deploy/debug exercise described below.

## Deploy Exercise

The application can’t be accessed after being deployed to an OpenShift cluster.  
Your task is to debug the deployment, identify the issue, and fix it so the application becomes reachable.

In this exercise, I’m using an OpenShift cluster via the [Red Hat Developer Sandbox](https://developers.redhat.com/developer-sandbox).

## Requirements

- Install the [OpenShift CLI (oc)](https://docs.okd.io/4.18/cli_reference/openshift_cli/getting-started-cli.html)
- Access to an OpenShift cluster (e.g. Red Hat Developer Sandbox)
- Connect the OpenShift CLI to the cluster:
  - Click on your username in the top-right corner in the OpenShift web console and select **“Copy login command”**
  - Paste and run the `oc login --token=... --server=...` command in your terminal
- Validate with `oc whoami` that the Openshift CLI is connected to the Openshift cluster

## Deploy the application

```bash
# You can create a new project or reuse an existing one, I'm using the default
# oc new-project hello-world

oc new-app golang~https://github.com/DanielBlei/hello-world.git

# Check deployment (it may take a few minutes to become Ready)
oc get pods
oc get deployments

# Check application logs (replace POD_NAME with the actual pod name)
oc logs -f POD_NAME

```

## Investigation Summary:

At first I checked the code quickly, built the Go package with `go build .`, and ran it to confirm it worked locally.

Once I deployed the application in OpenShift, I noticed a new Deployment with a single replica, after a few minutes the pod was running.

However, there is no Ingress, Service or Route to access the application.

### Vanilla K8s

Following my previous attempt, in a minikube or other clusters we need to expose the application via a **NodePort** or a **LoadBalancer** service, e.g:

```bash
kubectl expose deployment hello-world \
  --name=hello-world-svc \
  --port=8080 \
  --target-port=8080
  --type="NodePort"
```

Or even do a port forward from a ClusterIP service via `kubectl port-forward svc/hello-world-svc 8080:8080`

### OpenShift Approach

On OpenShift, using a Route is better because it builds on the platform’s built-in router and TLS/hostname management, so I only expose a normal ClusterIP Service and let OpenShift handle the heavy lift.

```bash
# Create a ClusterIP service
oc expose deployment hello-world \
  --name=hello-world-svc \
  --port=8080 \
  --target-port=8080

# Test Locally
oc port-forward svc/hello-world-svc 8080:8080

curl localhost:8080 # -> Hello OpenShift for Developers!
```

Now we have validated the connection locally, let's open it to others (be careful when allowing external access):

```bash
# We exposed the Deployment, now we need to expose the service
# Openshift will create a route object pointing to the desired service
oc expose svc/hello-world-svc --port=8080

# Get Endpoint access
oc get routes

curl YOUR_HOST_ENDPOINT
```

Thanks to the original [openshift-for-developers/hello](https://github.com/openshift-for-developers/hello) project for the base application used in this exercise.
