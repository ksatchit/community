# Setup kafka with KUDO

Notes: 
 - These steps were performed on a Konvoy cluster created on AWS
 - In this demo, we will use OpenEBS LocalPVs as the persistent storage for the kafka broker and zookeeper clusters.
   Ensure that you have installed OpenEBS on the cluster. For steps to do this, refer the [OpenEBS quickstart guide](https://docs.openebs.io/docs/next/quickstart.html)

   A quick step to install the OpenEBS control plane, default storage class & NDM amongst other components is provided below: 

   ```
   kubectl apply -f https://openebs.github.io/charts/openebs-operator-1.4.0.yaml

   ```

## Pre-requisites

- Setup a Kubernetes Cluster in version 1.13 or later
- Install kubectl version 1.13 or later

## Install KUDO CLI

### For Mac

```bash
brew tap kudobuilder/tap
brew install kudo-cli
```

### For Ubuntu

```bash
mkdir $GOPATH/src/github.com/kudobuilder
cd $GOPATH/src/github.com/kudobuilder
git clone https://github.com/kudobuilder/kudo.git
cd kudo/
make cli-install
```

## Install KUDO into your cluster

```bash
kubectl kudo init
```

## Installing the Operator

### Install Zookeeper

```bash
kubectl kudo install zookeeper --instance=zookeeper-instance
```

Note: In the demo cluster `openebs-hostpath` is set as the default storageclass. If it isn't, use the below command to explicitly set the same

```bash
kubectl kudo install zookeeper --instance=zookeeper-instance -p STORAGE_CLASS=openebs-hostpath
```

### Install Kafka

```bash
kubectl kudo install kafka --instance=kafka
```

Note: In the demo cluster `openebs-hostpath` is set as the default storageclass. If it isn't, use the below command to explicitly set the same

```bash
kubectl kudo install kafka --instance=kafka -p STORAGE_CLASS=openebs-hostpath
```

## Removing the Operator

```bash
kubectl delete instance kafka zookeeper-instance
kubectl delete crd operators.kudo.dev operatorversions.kudo.dev 
kubectl delete ns kudo-system
```

---
Note: Make sure you don't have any local folder with name kafka or zookeeper in the place where you are executing these commands
