# Accumulo in k8s proof of concept

This repo contains instructions and Kubernetes yaml descriptors for running a single node Accumulo cluster in Kubernetes. The original architecture for this deployment was Apache ZooKeeper, Apache Hadoop HDFS, and Accumulo running in Kubernetes. I could not find a good example for deploying HDFS in k8s, so I attempted to use Apache [Ozone](https://ozone.apache.org/) using the instructions at [here](https://ozone.apache.org/docs/1.2.1/start/minikube.html). I ran into some issues and eventually found [this](https://issues.apache.org/jira/browse/HDDS-6438) JIRA issue. At this point I started looking at alternatives for storage and looked at Ceph and MinIO. Reading the docs for Ceph it seemed like I would need a host with two disks and raw devices. Plus, it seemed a little complicated for a proof of concept.

>__Warning__
> This proof of concept was about getting Accumulo deployed on k8s. The setup below works, but Accumulo runs into issues with the write ahead logs as detailed [here](https://accumulo.apache.org/blog/2019/09/10/accumulo-S3-notes.html) and similarly with HBase on S3 [here](https://docs.aws.amazon.com/emr/latest/ReleaseGuide/emr-hbase-s3.html). The Hadoop [S3A](https://hadoop.apache.org/docs/stable/hadoop-aws/tools/hadoop-aws/index.html) filesystem implementation does not support sync as messages like `WARN s3a.S3ABlockOutputStream: Application invoked the Syncable API against stream writing to <wal file>. This is unsupported` appear in the logs.

# Setup

## Prerequisites

To run this proof of concept you need [Docker](https://docs.docker.com/get-docker/), [KIND](https://kind.sigs.k8s.io/docs/user/quick-start/#installation), [kubectl](https://kubernetes.io/docs/tasks/tools/), and [Helm](https://helm.sh/docs/intro/install/) installed.

## Creating a KIND cluster
```bash
kind create cluster
```

At this point you should be able to run the following to see pods running
```bash
kubectl get pods -n kube-system
```

## Installing ZooKeeper

Do the following to install ZooKeeper via a Helm chart:
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
kubectl create namespace zookeeper
helm install --namespace zookeeper bitnami-zookeeper bitnami/zookeeper
```

>__Note__
> If you want to connect to ZooKeeper you can do the following:
> ```bash
> kubectl port-forward --namespace zookeeper svc/bitnami-zookeeper 2181:2181 &
> zkCli.sh 127.0.0.1:2181
> ```
>
> If you want to check the ZooKeeper logs you can do the following:
> ```bash
> kubectl logs -f $(kubectl get pod -n zookeeper -l app.kubernetes.io/name=zookeeper -o jsonpath="{.items[0].metadata.name}") -n zookeeper
> ```

## Installing MinIO

The instructions for installing MinIO are [here](https://docs.min.io/minio/baremetal/quickstart/k8s.html). I did the following on my machine:

```bash
sudo mkdir /export
sudo sudo chown "`id -un`.`id -gn`" /export
curl https://raw.githubusercontent.com/minio/docs/master/source/extra/examples/minio-dev.yaml -O
modified minio-dev.yaml:
  - comment out node selector
  - fix spec.volumes.hostPath.path to point to /export
kubectl apply -f minio-dev.yaml
kubectl apply -f minio-svc.yaml
```

### Configuring MinIO

You need to configure MinIO using the web console. Do the following:

  1. `kubectl port-forward -n minio-dev pod/minio 9000 9090 &`
  2. Log into console at `http://localhost:9090` using the username `minioadmin` and password `minioadmin`
  3. Create a user in the web console
     1. On the left menu go to Identity -> Users, then click the `Create User` button on the right
     2. Use the username `accumulo` and the password `changeme`, and set the policy to `readwrite`, then click Save
  4. Create a bucket
     1. On the left menu go to Buckets, then click the `Create Bucket` button on the right
     2. Use the name `accumulo`, then click `Create Bucket`  

## Installing Accumulo

For this stage you will need 3 git repositories checked out:

  1. [accumulo](https://github.com/apache/accumulo)
  2. [accumulo-docker](https://github.com/apache/accumulo-docker)
  3. This repo

### Accumulo Docker image

In the accumulo repo, run:
```bash
git checkout main
mvn clean package -DskipTests
```

Then copy the accumulo binary tarball from the assemble/target directory to the accumulo-docker repo directory. In the accumulo-docker repo, run
```bash
git checkout next-release
docker build --build-arg ACCUMULO_FILE=accumulo-2.1.0-SNAPSHOT-bin.tar.gz -t accumulo:2.1.0 .
```

### Build Accumulo-S3-fs Docker image

Build the accumulo-s3-fs image from this directory and load the image into the Kubernetes environment, run:
```bash
cd docker
docker build -t accumulo-s3-fs:2.1.0 .
kind load docker-image accumulo-s3-fs:2.1.0
```

### Create the Accumulo Deployment

  1. Create the kubernetes accumulo namespace: `kubectl apply -f accumulo-ns.yaml`
  2. Create the Accumulo secrets object: `kubectl apply -f accumulo-secrets.yaml`
  3. Create the Accumulo configuration objects: `kubectl apply -f accumulo-config.yaml`
  4. Initialize Accumulo: `kubectl apply -f accumulo-init.yaml`
     1. Check that the job succeeded successfully:
        ```bash
        kubectl logs -f $(kubectl get pod -n accumulo -l job-name=job -o jsonpath="{.items[0].metadata.name}") -n accumulo
        ```
     2. Remove the job
        ```bash
        kubectl delete -f accumulo-init.yaml
        ```
  5. Deploy the Accumulo server processes:
     ```
     kubectl apply -f accumulo-manager.yaml 
     kubectl apply -f accumulo-gc.yaml 
     kubectl apply -f accumulo-tserver.yaml 
     kubectl apply -f accumulo-monitor.yaml
     ```
  6. Optionally, deploy the optional Accumulo server processes:
     ```
     kubectl apply -f accumulo-coordinator.yaml
     kubectl apply -f accumulo-compactor.yaml
     kubectl apply -f accumulo-sserver.yaml
     ```
>__Note__
> If you want to view the Accumulo Monitor:
> ```bash
> kubectl port-forward deployment/accumulo-monitor -n accumulo 9995:9995 &
> ```
> 
> and point your browser at `localhost:9995`


# TearDown Everything

```
kubectl delete -f accumulo-coordinator.yaml
kubectl delete -f accumulo-compactor.yaml
kubectl delete -f accumulo-sserver.yaml
kubectl delete -f accumulo-manager.yaml
kubectl delete -f accumulo-gc.yaml
kubectl delete -f accumulo-tserver.yaml
kubectl delete -f accumulo-monitor.yaml
kubectl delete -f accumulo-config.yaml
kubectl delete -f accumulo-secrets.yaml
kubectl delete -f accumulo-ns.yaml
kubectl delete -f minio-svc.yaml
kubectl delete -f minio-dev.yaml
helm uninstall --namespace zookeeper bitnami-zookeeper
kubectl delete namespace zookeeper
kind delete cluster
sudo rmdir /export
```

