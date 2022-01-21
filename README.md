A Kafka / Confluent GitOps workflow example for multi-env deployments with Flux, Kustomize, Helm and Confluent Operator.

## Usage

For this example we assume a single cluster simulating a production confluent environment. The end goal is to leverage Flux and Kustomize to manage [Confluent Operator for Kubernetes](https://github.com/confluentinc/operator-earlyaccess). You can extend with another cluster while minimizing duplicated declarations.

We will configure [Flux](https://fluxcd.io/) to install, deploy and config the [Confluent Platform](https://www.confluent.io/product/confluent-platform) using their `HelmRepository` and `HelmRelease` custom resources.
Flux will monitor the Helm repository, and can be configured to automatically upgrade the Helm releases to their latest chart version based on semver ranges.


## Prerequisites

A CNCF compatible Kubernetes cluster.


## Repository structure

The Git repository contains the following top directories:

- **flux-system** dir contains the required kubernetes resources for flux to operate
- **kustomize/base** dir contains the base definition of the confluent stack.
- **kustomize/environments** dir containing an example environment, folders could be copied to create additional environments.  Files within are 'patches' which are layered on top of the definitions found in kustomize/base
- **kustomize/operator** dir the helm chart definition for confluent-for-kubernetes (CFK).


```
├── flux-system
├── kustomize
│   ├── base
│   │   ├── confluent
│   ├── environments
│   │   └── sandbox
│   └── operator
```

## Initial scaffold
In order to showcase the GitOps behaviour of the FluxCD toolkit you will require the ability to write to a repository.  Fork this repository, and update line 11 of the file `./flux-system/gotk-sync.yaml` to the new https git address of your forked repository.  Also make note of line 10 'branch'; this is the branch of the repository which Flux will monitor actively.

### Deploy base Flux components
#### Overview
This step will install the base Flux kubernetes components onto your kubernetes cluster.  To inspect what is being applied, simply look through the contents of `./flux-system/gotk-components.yaml`.  You will see a mix of Custom Resource Definitions, Service Accounts, Deployments, and other various components.  After the application of these resource definitions is completed, you should see the following pods running:

* Helm-Controller
* Kustomize Controller
* Notification Controller
* Source Controller

For more information on what these controllers do, please review [the documentation here](https://fluxcd.io/docs/components/).



### Deployment Process
* Navigate to `./flux-system`
* Run `kubectl apply -f gotk-components.yaml`


### Deploy Flux Sync
#### Overview
This next step will tell Flux what repository to monitor, and, within that repository, what kustomization files to start with.  The first Kustomize resource that Flux will look for to is located at `./kustomize/operator`.  This will install the confluent-for-kubernetes Helm chart and nginx controller.  

### Deployment Process
* Navigate to `./flux-system`
* run `kubectl apply -f gotk-sync.yaml`

### Some customization parameters

1. The Confluent operator only manages Confluent Kafka clusters in the specific namespace it is installed in. This behaviour is overridden where we've installed a global per cluster Confluent operator pod in `kustomize/operator/confluent-operator-helm-release-confluent.yaml` line #38.

```
    namespaced: false
```

2. The Confluent operator is installed in an `operators` namespace where conventionally other operators are installed. This can be changed in `kustomize/operator/namespaces.yaml` file and `kustomize/operator/confluent-operator-helm-release-confluent.yaml` line #14.

```
  namespace: operators
```

3. Nginx controller is installed in default namespace. We will be provisioning a cluster with static host based access.
In many load balancer use cases, the decryption happens at the load balancer, and then unencrypted data is passed along to the endpoint. This is known as SSL termination. With the Kafka protocol, however, the broker expects to perform the SSL handshake directly with the client. To achieve this, SSL passthrough is required. SSL passthrough is the action of passing data through a load balancer to a server without decrypting it. Therefore, we configure this parameter in the nginx controller installation in file `kustomize/operator/nginx-controller-helm-release.yaml` line #35:

```
  values:
    controller.extraArgs.enable-ssl-passthrough: true
```

## Sandbox cluster

 After a successful installation of Confluent operator and Nginx controller, we will proceed to deploy our first environment located at  `./kustomize/environments/sandbox`.


### Prerequisites to setup sandbox cluster

1. Portworx storage class
2. A cluster level domain name
3. Setting up of mTLS related secrets


### Deployment Process
* Navigate to `./flux-system`
* run `kubectl apply -f sandbox-cluster.yaml`

### Demostrating Gitops of the installed cluster

Now that we have flux monitoring the forked Git repository, let's demonstrate the GitOps behaviour!  If everything has deployed successfully, you should see a healthy confluent stack looking like this:
```console
$ kubectl get pod
NAME                                        READY   STATUS     RESTARTS   AGE
confluent-operator-7cd67bfdc5-gxfdx         1/1     Running    0          37h
connect-0                                   0/1     Running    0          44s
connect-1                                   1/1     Running    0          6m53s
controlcenter-0                             1/1     Running    0          4h38m
ingress-nginx-controller-5bb9984b7b-dbj45   1/1     Running    0          106m
kafka-0                                     1/1     Running    0          58m
kafka-1                                     1/1     Running    0          11h
kafka-2                                     0/1     Running    0          11h
ksqldb-0                                    1/1     Running    0          148m
ksqldb-1                                    1/1     Running    0          151m
schemaregistry-0                            1/1     Running    0          13h
zookeeper-0                                 1/1     Running    0          14h
zookeeper-1                                 1/1     Running    0          14h
zookeeper-2                                 1/1     Running    0          14h
```
To exhibit Flux, let's change our kafka replicas from the default of 3, to 4:
* In the file `./kustomize/environments/sandbox/kafka.yaml` uncomment the line `#  replicas: 4`, commit that change to your repository (git), and push upstream.   The next time flux performs a 'sync' (observable in the 'source controller' logs), it will the change to the kafka spec, and in turn increase our kafka cluster from size '3' to '4'.


## Simple producer and consumer from the CLI

## Kafka connect

### How to configure connect plugins

### How to add new topic

### How to add new connector

## Schema registry

## KSQL

## Destroy the cluster

## Need help/customization

File a GitHub [issue](https://github.com/Platformatory/confluent-kubernetes-bootstrap/issues), send us an [email](mailto:pavan@platformatory.com).

## Legalese

Copyright © 2022 [Platformatory](https://platformatory.io/) | See [LICENCE](LICENSE) for full details.
