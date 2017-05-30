---
date: 2017-05-25T14:38:08-07:00
subtitle: A simple guide to create an Azure Container Service Kubernetes cluster and deploying Istio to it.
tags: [azure, kubernetes, containers, acs, k8s]
title: Deploying Istio on Azure Container Service
---
Yesterday, IBM, Google and Lyft [announced Istio](https://developer.ibm.com/dwblog/2017/istio/), "an open technology that provides a way for developers to seamlessly connect, manage and secure networks of different microservices—regardless of platform, source or vendor." 
<!--more-->
In this tutorial I will be showing you how to deploy Istio to a new Kubernetes cluster in Azure Container Service.

## Creating an Azure Container Service Kubernetes Cluster

#### A few prerequisites:

- Use BASH, ZSH or other shells with BASH-compatibility
- Use the Azure Cloud Shell in the [Azure Portal](https://portal.azure.com), or make sure to install the [Azure CLI 2.0](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)


```bash
# Creates a resource group
#
# Note, ACS is available several in other locations,
# but we will use US West for this tutorial.
az group create --name <ANY-GROUP-NAME> --location=westus
```

```bash
# Creates a Kubernetes cluster on Azure Container Service
az acs create --orchestrator-type=kubernetes \
--resource-group <ANY-GROUP-NAME> --name=<ANY-CLUSTER-NAME> \
--dns-prefix=<ANY-PREFIX> --generate-ssh-keys
```

## Configure your Kubernetes client

If you are not using the Azure Cloud Shell and don't have the Kubernetes client `kubectl`, run `sudo az acs kubernetes install-cli`.

```bash
# Copies the Kubernetes config from the Kubernetes master node
scp azureuser@<ANY-PREFIX>.westus.cloudapp.azure.com:.kube/config \
$HOME/.kube/config
```

You can verify your Kubernetes credentials by printing the Kubernetes server information.

```bash
kubectl version
```

## Installing Istio

In your working directory, run the following command to [download the latest Istio release](https://github.com/istio/istio/releases).

```bash
curl -L https://git.io/getIstio | sh -
```


### Option 1: Install using the Helm package manager

If you don't already have the Helm package manager for Kubernetes, make sure to [install](https://docs.helm.sh/using-helm/#install-helm) it. Since I am using `homebrew` on a Mac I did so via `brew install kubernetes-helm`.

Configure your Kubernetes client by running the following command.
```bash
# Sets up Helm according to our Kubernetes config file
helm init
```

Now we add the experimental `incubator` repos.
```bash
helm repo add incubator \
https://kubernetes-charts-incubator.storage.googleapis.com/
helm repo update
```

Finally, we install the *Istio* incubator package with RBAC disabled. *Azure Container Service does not have RBAC support enabled at this time*

```
helm install incubator/istio --set rbac.install=false
```

### Option 2: Manual install

Navigate into the Istio folder and run

```bash
kubectl apply -f install/kubernetes/istio-auth.yaml
kubectl apply -f install/kubernetes/addons/prometheus.yaml
kubectl apply -f install/kubernetes/addons/grafana.yaml
kubectl apply -f install/kubernetes/addons/servicegraph.yaml
kubectl apply -f install/kubernetes/addons/zipkin.yaml
```

### Verify your installation

Grafana is now running. Get the public IP address to access Grafana in your browser.

```bash
export PODNAME=$(kubectl get pods | grep "grafana" | awk '{print $1}')
kubectl port-forward $PODNAME 3000:3000
```

Navigate to `http://localhost:3000/dashboard/db/istio-dashboard`

![Creating a CDN](/img/deploying-istio/grafana.png)

## Installing the Istio sample application

Install the Istio "bookinfo" sample application by running

```bash
kubectl apply -f <(istioctl kube-inject -f \
samples/apps/bookinfo/bookinfo.yaml)
```

Get the ingress public IP address for your Kubernetes cluster
```
kubectl get services | grep "ingress" | awk '{print $3}'
```

Using the IP address from the above command, generate some traffic to the sample application.
```bash
curl -I http://{INGRESS_IP_ADDRESS}/productpage
```

Finally, you can view a diagram of traffic flow for the sample application by accessing the servicegraph addon service in your browser.

```bash
export PODNAME=$(kubectl get pods | grep "servicegraph" | awk '{print $1}')
kubectl port-forward $PODNAME 8088:8088
```

Navigate to `http://localhost:8088/dotviz`

![Creating a CDN](/img/deploying-istio/servicegraph.png)

### Additional Resources

- [Azure Documentation: Get started with a Kubernetes Cluster in Container Service](https://docs.microsoft.com/en-us/azure/container-service/container-service-kubernetes-walkthrough)
- [Istio Documentation](https://istio.io/docs/)