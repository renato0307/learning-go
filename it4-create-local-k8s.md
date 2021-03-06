# Create a local kubernetes cluster

__What is Kubernetes?__
[Kubernetes](https://kubernetes.io/), also known as K8s, is an open-source
system for automating deployment, scaling, and management of containerized
applications.

I'm not going to deep here on the k8s details as there are plenty resources
available to achieve that purpose.

Were are going to use [Kind](https://kind.sigs.k8s.io/) to run Kubernetes
locally:

> Kind is a tool for running local Kubernetes clusters using Docker container 
> βnodesβ. kind was primarily designed for testing Kubernetes itself, but may be
> used for local development or CI.

To install it follow the instructions at
https://kind.sigs.k8s.io/docs/user/quick-start/#installation.

If you're using Windows and WSL2 check the specific instructions at
https://kind.sigs.k8s.io/docs/user/using-wsl2.

You'll also need to install kubectl, k8s command-line tool, that allows you to
run commands against Kubernetes clusters. Installation instructions can be found
at https://kubernetes.io/docs/tasks/tools.

If you didn't create a k8s cluster during the Kind installation do it now:

```sh
kind create cluster
```

The output should be similar to:

```
Creating cluster "kind" ...
 β Ensuring node image (kindest/node:v1.21.1) πΌ
 β Preparing nodes π¦
 β Writing configuration π
 β Starting control-plane πΉοΈ
 β Installing CNI π
 β Installing StorageClass πΎ
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Not sure what to do next? π  Check out https://kind.sigs.k8s.io/docs/user/quick-start
```

Additionally we would also need to have the service of type
[LoadBalancer](https://kubernetes.io/docs/tasks/access-application-clustercreate-external-load-balancer/) working on the 
cluster.

Follow the instructions available at https://kind.sigs.k8s.io/docs/user/loadbalancer.

# Next
 
The next section is
[Create a Helm chart for the API](it4-create-helm-chart-api.md).