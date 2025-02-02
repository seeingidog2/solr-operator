# Solr Operator
[![Build Status](https://travis-ci.com/bloomberg/solr-operator.svg?branch=master)](https://travis-ci.com/bloomberg/solr-operator) [![Go Report Card](https://goreportcard.com/badge/github.com/bloomberg/solr-operator)](https://goreportcard.com/report/github.com/bloomberg/solr-operator) ![Latest Version](https://img.shields.io/github/tag/bloomberg/solr-operator) [![Docker Pulls](https://img.shields.io/docker/pulls/bloomberg/solr-operator)](https://hub.docker.com/r/bloomberg/solr-operator/)

The __Solr Operator__ manages Apache Solr Clouds within Kubernetes. It is built on top of the [Kube Builder](https://github.com/kubernetes-sigs/kubebuilder) framework.

The project is currently in beta (`v1beta1`), and while we do not anticipate changing the API in backwards-incompatible ways there is no such guarantee yet.

## Menu

- [Getting Started](#getting-started)
- [Contributions](#contributions)
- [License](#license)
- [Code of Conduct](#code-of-conduct)
- [Security Vulnerability Reporting](#security-vulnerability-reporting)

## Getting Started

### Running the Solr Operator

Install the Zookeeper & Etcd Operators, which this operator depends on by default.

```bash
$ kubectl apply -f example/ext_ops.yaml
```

Install the SolrCloud CRD & Operator

```bash
$ kubectl apply -f config/crds/solr_v1beta1_solrcloud.yaml
$ kubectl apply -f config/operators/solr_operator.yaml
```
                        
### Lifecyle of a Solr Cloud

#### Creating

Make sure that the solr-operator and a zookeeper-operator are running.

Create an example Solr cloud, with the following configuration.

```bash
$ cat example/test_solrcloud.yaml

apiVersion: solr.bloomberg.com/v1beta1
kind: SolrCloud
metadata:
  name: example
spec:
  replicas: 4
  solrImage:
    tag: 8.1.1
```

Apply it to your Kubernetes cluster.

```bash
$ kubectl apply -f example/test_solrcloud.yaml
$ kubectl get solrclouds

NAME      VERSION   DESIREDNODES   NODES   READYNODES   AGE
example   8.1.1     4              2       1            2m

$ kubectl get solrclouds

NAME      VERSION   DESIREDNODES   NODES   READYNODES   AGE
example   8.1.1     4              4       4            8m
```

#### Scaling

Increase the number of Solr nodes in your cluster.

```bash
$ kubectl scale --replicas=5 solrcloud/example
```

#### Deleting

Decrease the number of Solr nodes in your cluster.

```bash
$ kubectl delete solrcloud example
```

#### Dependent Kubernetes Resources

What actually gets created when the Solr Cloud is spun up?

```bash
$ kubectl get all

NAME                                       READY   STATUS             RESTARTS   AGE
pod/example-solrcloud-0                    1/1     Running            7          47h
pod/example-solrcloud-1                    1/1     Running            6          47h
pod/example-solrcloud-2                    1/1     Running            0          47h
pod/example-solrcloud-3                    1/1     Running            6          47h
pod/example-solrcloud-zk-0                 1/1     Running            0          49d
pod/example-solrcloud-zk-1                 1/1     Running            0          49d
pod/example-solrcloud-zk-2                 1/1     Running            0          49d
pod/example-solrcloud-zk-3                 1/1     Running            0          49d
pod/example-solrcloud-zk-4                 1/1     Running            0          49d
pod/solr-operator-8449d4d96f-cmf8p         1/1     Running            0          47h
pod/zk-operator-674676769c-gd4jr           1/1     Running            0          49d

NAME                                       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)               AGE
service/example-solrcloud-0                ClusterIP   ##.###.###.##    <none>        80/TCP                47h
service/example-solrcloud-1                ClusterIP   ##.###.##.#      <none>        80/TCP                47h
service/example-solrcloud-2                ClusterIP   ##.###.###.##    <none>        80/TCP                47h
service/example-solrcloud-3                ClusterIP   ##.###.##.###    <none>        80/TCP                47h
service/example-solrcloud-common           ClusterIP   ##.###.###.###   <none>        80/TCP                47h
service/example-solrcloud-headless         ClusterIP   None             <none>        80/TCP                47h
service/example-solrcloud-zk-client        ClusterIP   ##.###.###.###   <none>        21210/TCP             49d
service/example-solrcloud-zk-headless      ClusterIP   None             <none>        22210/TCP,23210/TCP   49d

NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/solr-operator              1/1     1            1           49d
deployment.apps/zk-operator                1/1     1            1           49d

NAME                                       DESIRED   CURRENT   READY   AGE
replicaset.apps/solr-operator-8449d4d96f   1         1         1       2d1h
replicaset.apps/zk-operator-674676769c     1         1         1       49d

NAME                                       READY   AGE
statefulset.apps/example-solrcloud         4/4     47h
statefulset.apps/example-solrcloud-zk      5/5     49d

NAME                                          HOSTS                                                                                       PORTS   AGE
ingress.extensions/example-solrcloud-common   default-example-solrcloud.test.domain,default-example-solrcloud-0.test.domain + 3 more...   80      2d2h

NAME                                       VERSION   DESIREDNODES   NODES   READYNODES   AGE
solrcloud.solr.bloomberg.com/example       8.1.1     4              4       4            47h
```

### Solr Backups

Solr backups require 3 things:
- A solr cloud running in kubernetes to backup
- The list of collections to backup
- A shared volume reference that can be written to from many clouds
    - This could be a NFS volume, a persistent volume claim (that has `ReadWriteMany` access), etc.
    - The same volume can be used for many solr clouds in the same namespace, as the data stored within the volume is namespaced.
- A way to persist the data. The currently supported persistence methods are:
    - A volume reference (this does not have to be `ReadWriteMany`)
    - An S3 endpoint.
    
Backups will be tarred before they are persisted.

There is no current way to restore these backups, but that is in the roadmap to implement.
  
  
## Solr Images

The solr-operator will work with any of the [official Solr images](https://hub.docker.com/_/solr) currently available.

## Zookeeper

Solr Clouds require an Apache Zookeeper to connect to.

The Solr operator gives a few options.

### ZK Connection Info

This is an external/internal connection string as well as an optional chRoot to an already running Zookeeeper ensemble.
If you provide an external connection string, you do not _have_ to provide an internal one as well.

### Provided Instance

If you do not require the Solr cloud to run cross-kube cluster, and do not want to manage your own Zookeeper ensemble,
the solr-operator can manage Zookeeper ensemble(s) for you.

#### Zookeeper

Using the [zookeeper-operator](https://github.com/pravega/zookeeper-operator), a new Zookeeper ensemble can be spun up for 
each solrCloud that has this option specified.

The startup parameter `zookeeper-operator` must be provided on startup of the solr-operator for this parameter to be available.

#### Zetcd

Using [etcd-operator](https://github.com/coreos/etcd-operator), a new Etcd ensemble can be spun up for each solrCloud that has this option specified.
A [Zetcd](https://github.com/etcd-io/zetcd) deployment is also created so that Solr can interact with Etcd as if it were a Zookeeper ensemble.

The startup parameter `etcd-operator` must be provided on startup of the solr-operator for this parameter to be available.

## Solr Operator

### Solr Operator Input Args

* **-zookeeper-operator** Whether or not to use the Zookeeper Operator to create dependent Zookeeepers.
                          Required to use the `ProvidedZookeeper.Zookeeper` option within the Spec.
                          If _true_, then a Zookeeper Operator must be running for the cluster.
                          ( _true_ | _false_ , defaults to _false_)
* **-etcd-operator** Whether or not to use the Etcd Operator to create dependent Zetcd clusters.
                     Required to use the `ProvidedZookeeper.Zetcd` option within the Spec.
                     If _true_, then an Etcd Operator must be running for the cluster.
                     ( _true_ | _false_ , defaults to _false_)
* **-ingress-base-url** If you desire to make solr externally addressable via ingresses, a base ingress domain is required.
                        Solr Clouds will be created with ingress rules at `*.(ingress-base-url)`.
                        ( _optional_ , e.g. `ing.base.domain` )
## Development

### Updating the CRD

The CRD should be updated anytime you update the API.

```bash
$ make manifests
```

### Docker

#### Docker Images

Two Docker images are published to [DockerHub](https://hub.docker.com/r/bloomberg/solr-operator), both based off of the same base image.

- [Base Image](build/Dockerfile.build.dynamic) - Downloads vendor directories, builds operator executable (This is not published, only used to build the following images)
- [Slim Image](build/Dockerfile.slim) - Contains only the operator executable
- [Vendor Image](build/Dockerfile.slim) - Contains the operator executable as well as all vendored dependencies (at `/solr-operator-vendor-sources`)

#### Building

Building and releasing a test operator image with a custom namespace.

```bash
$ NAMESPACE=your-namespace make docker-base-build docker-build docker-push
```

## Contributions

We :heart: contributions.

Have you had a good experience with <project>? Why not share some love and contribute code, or just let us know about any issues you had with it?

We welcome issue reports [here](../../../issues); be sure to choose the proper issue template for your issue, so that we can be sure you're providing the necessary information.

Before sending a [Pull Request](../../../pulls), please make sure you read our
[Contribution Guidelines](https://github.com/bloomberg/.github/blob/master/CONTRIBUTING.md).

## License

Please read the [LICENSE](../LICENSE) file here.

## Code of Conduct

This project has adopted a [Code of Conduct](https://github.com/bloomberg/.github/blob/master/CODE_OF_CONDUCT.md).
If you have any concerns about the Code, or behavior which you have experienced in the project, please
contact us at opensource@bloomberg.net.

## Security Vulnerability Reporting

If you believe you have identified a security vulnerability in this project, please send email to the project
team at opensource@bloomberg.net, detailing the suspected issue and any methods you've found to reproduce it.

Please do NOT open an issue in the GitHub repository, as we'd prefer to keep vulnerability reports private until
we've had an opportunity to review and address them.
