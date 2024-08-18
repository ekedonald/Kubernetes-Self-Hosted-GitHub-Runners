# Kubernetes Self-Hosted GitHub Runners

This guide will walk you through the process of installing and configuring a Self-Hosted GitHub Actions Runner on a Kubernetes Cluster.

### Prerequisites
- **Repository Access**: Create a GitHub Organization and fork one of your repos to it.
- **Kubernetes Cluster**: Use an existing Kubernetes cluster or set up a new one ([Minikube](https://minikube.sigs.k8s.io/docs/start/?arch=%2Fmacos%2Fx86-64%2Fstable%2Fbinary+download), GKE, EKS or AKS).
- **Kubectl**: Install the Kubernetes command-line tool [kubectl](https://kubernetes.io/docs/tasks/tools/) and configure it to connect to your cluster.
- **Helm**: Install [Helm](https://helm.sh/docs/intro/install/), the Kubernetes package manager.

_**Note:** If you are using Minikube, install Docker and select it as your preferred driver._

The following steps are taken to install a Self-Hosted GitHub Actions Runner on a Kubernetes Cluster:

## Step 1: Install Cert-Manager

The Cert-Manager is required before installing a runner because it plays a crucial role in securing the communication within the Kubernetes cluster, especially for the Actions Runner Controller (ARC) components. It is vital for secure communication, webhook validation, component authentication and automated certificate lifecycle.

- Run the following command to install Cert-Manager on your cluster.

```sh
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.8.2/cert-manager.yaml
```

## Step 2: Authenticate Action Runner Controller with a Personal Access Token (Classic)

- Create a personal access token (classic) with the following required scopes:

  - **Repository runners**: `repo`
  - **Organization runners**: `admin:org`

- Copy the generated personal access token, it will be used when installing the Actions Runner Controller.

## Step 3: Install Actions Runner Controller using Helm

The command below uses Helm to install the Actions Runner Controller in a Kubernetes Cluster. It sets up the necessary components in a specified namespace, configures authentication with GitHub and waits for the installation to complete before finishing. This controller allows you to run GitHub Actions runners in your Kubernetes cluster.

- Insert your `personal access token` before executing the command.

```sh
helm repo add actions-runner-controller https://actions-runner-controller.github.io/actions-runner-controller
helm upgrade --install --namespace actions-runner-system --create-namespace \
             --set=authSecret.create=true \
             --set=authSecret.github_token="YOUR_GITHUB_TOKEN" \
             --wait actions-runner-controller actions-runner-controller/actions-runner-controller
```

- This will give you an output:

```sh
kubectl --namespace actions-runner-system port-forward $POD_NAME 8080:$CONTAINER_PORT
```

- Head to your browser and enter `localhost:8080`. This will display **Client sent an HTTP request to an HTTPS server**.

## Step 4: Create a Runner Deployment Manifest

- Create a file `runner.yaml` and insert your **organization name**, **your repo** and **runner label** in the repository context.

```sh
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: telex-runner
spec:
  replicas: 1
  template:
    spec:
      repository: your-org/your-repo
      labels:
      - Telex
```

- Apply the configuration.

```sh
kubectl apply -f runner.yaml
```

- Verify if the pod is running

```sh
kubectl get pods -w
```

## Step 5: Configure your Workflow to use a Self-Hosted Runner
- Go to your repository on GitHub and modify the runner to use `Self-Hosted` or the `label` (_i.e. Telex_) you indicated in the Runner Deployment manifest.

```sh
name: Build Binary and Run Tests

on: workflow_dispatch

jobs:
  build:
    runs-on: Telex
```

- Since the workflow above is triggered manually, trigger it and wait for the job to build.

![runner triggered](./images/1%20runner%20trigger.png)
![runner info](./images/3%20runner%20info.png)

- Run the following command to check the logs of the runner pods:

```sh
kubectl logs -f <POD_NAME>
```

![pod logs](./images/2%20pod%20logs.png)
![completed job](./images/completed%20job.png)
_The Self-Hosted GitHub Runner is fully functional._
